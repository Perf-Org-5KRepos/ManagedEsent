# SystemStats Sample
This shows how to use a PersistentDictionary to build a simple application to track and report on some performance counters. There are two main parts:
1. A mode that periodically collects performance counter values and stores them.
2. A mode that queries the stored data.

## The Sample structure
This program will use a PersistentDictionary to map a DateTime to a collection of samples. We will group the samples together into a serializable structure.

```c#
/// <summary>
/// One sample of the statistics we are collecting. This struct is stored in
/// the PersistentDictionary, keyed by time.
/// </summary>
[Serializable](Serializable)
internal struct Sample
{
    /// <summary>
    /// The processor usage.
    /// </summary>
    private float processorTime;

    /// <summary>
    /// The amount of system paging.
    /// </summary>
    private float paging;

    /// <summary>
    /// The amount of free memory.
    /// </summary>
    private float freeBytes;

    /// <summary>
    /// Gets or sets the processor usage.
    /// </summary>
    public float ProcessorTime
    {
        get { return this.processorTime; }
        set { this.processorTime = value; }
    }

    /// <summary>
    /// Gets or sets the amount of system paging.
    /// </summary>
    public float Paging
    {
        get { return this.paging; }
        set { this.paging = value; }
    }

    /// <summary>
    /// Gets or sets the amount of free memory.
    /// </summary>
    public float FreeBytes
    {
        get { return this.freeBytes; }
        set { this.freeBytes = value; }
    }
}
```

## Collecting Performance Counters

To collect performance counters we just add a new Sample to the dictionary. A PersistentDictionary is backed by an ESENT database which will always be consistent after a crash. That means we don't need to worry about data consistency after a crash.
 
```c#
/// <summary>
/// Collect stats. This method never returns (just kill the program).
/// </summary>
/// <param name="dictionaryPath">The path to the dictionary.</param>
private static void CollectStats(string dictionaryPath)
{
    using (PerformanceCounter processor = new PerformanceCounter("Processor", "% Processor Time", "_Total"))
    using (PerformanceCounter paging = new PerformanceCounter("Memory", "Pages/Sec"))
    using (PerformanceCounter freeMemory = new PerformanceCounter("Memory", "Available Bytes"))
    using (PersistentDictionary<DateTime, Sample> dictionary = new PersistentDictionary<DateTime, Sample>(dictionaryPath))
    {
        while (true)
        {
            dictionary[DateTime.Now](DateTime.Now) = new Sample
                {
                    ProcessorTime = processor.NextValue(),
                    Paging = paging.NextValue(),
                    FreeBytes = freeMemory.NextValue(),
                };

            Thread.Sleep(TimeSpan.FromSeconds(0.5));
        }
    }
}
```

Updates to a PersistentDictionary are logged lazily -- after a crash some updates may be lost. It is possible to force the updates to be written to disk by calling the Flush() method on the PersistentDictionary, but that has to perform an I/O so it is considerably slower than a lazy update. In this case we are willing to lose a few updates after a crash.

## Calculating statistics

While querying the data we will want to calculate the min, max and average of the samples. This class will be used to do the calculations:

```c#
/// <summary>
/// Calculates the average, min and max of a set of samples.
/// </summary>
private class Aggregator
{
    /// <summary>
    /// Number of samples seen.
    /// </summary>
    private int numSamples = 0;

    /// <summary>
    /// Running total of all the samples.
    /// </summary>
    private double total = 0;

    /// <summary>
    /// The smallest sample seen.
    /// </summary>
    private double min = Double.MaxValue;

    /// <summary>
    /// The largest sample seen.
    /// </summary>
    private double max = Double.MinValue;

    /// <summary>
    /// Gets the number of samples that were seen.
    /// </summary>
    public int NumSamples
    {
        get { return this.numSamples; }
    }

    /// <summary>
    /// Gets the average of all the samples.
    /// </summary>
    private double Average
    {
        get { return 0 == this.numSamples ? 0 : this.total / this.numSamples; }
    }

    /// <summary>
    /// Add a new sample.
    /// </summary>
    /// <param name="sample">The sample value/</param>
    public void AddSample(double sample)
    {
        this.numSamples++;
        this.total += sample;
        this.min = Math.Min(this.min, sample);
        this.max = Math.Max(this.max, sample);
    }

    /// <summary>
    /// Gets a string representation of the aggregate calculation.
    /// </summary>
    /// <returns>
    /// A string representation of the aggregate calculation
    /// </returns>
    public override string ToString()
    {
        return String.Format("min = {0:N2}, max = {1:N2}, average = {2:N2}", this.min, this.max, this.Average);
    }
}
```

## Querying the PersistentDictionary

The PersistentDictionary supports LINQ queries. Given a query like this:

```c#
DateTime startTime = ...;
DateTime endTime = ...;
IEnumerable<Sample> samples = from x in dictionary where x.Key >= startTime && x.Key <= endTime select x.Value;
```

Only records which match the key criteria will be retrieved from the database. That means the runtime is determined by the number of samples retrieved, not the total number of samples. For best performance we only want to iterate over the samples once so we accumulate all the statistics at the same time, using the Aggregator class:

```c#
/// <summary>
/// Dump the aggregate stats for the specified time period.
/// </summary>
/// <param name="dictionaryPath">The path to the dictionary.</param>
/// <param name="startTime">The starting time.</param>
/// <param name="endTime">The ending time.</param>
private static void DumpStats(string dictionaryPath, DateTime startTime, DateTime endTime)
{
    if (!PersistentDictionaryFile.Exists(dictionaryPath))
    {
        Console.WriteLine("No stats collected");
        return;
    }

    using (PersistentDictionary<DateTime, Sample> dictionary = new PersistentDictionary<DateTime, Sample>(dictionaryPath))
    {
        Stopwatch queryTimer = Stopwatch.StartNew();
        IEnumerable<Sample> samples = from x in dictionary where x.Key >= startTime && x.Key <= endTime select x.Value;

        Aggregator processor = new Aggregator();
        Aggregator paging = new Aggregator();
        Aggregator freeBytes = new Aggregator();
        foreach (Sample s in samples)
        {
            processor.AddSample(s.ProcessorTime);
            paging.AddSample(s.Paging);
            freeBytes.AddSample(s.FreeBytes);
        }

        queryTimer.Stop();
        Console.WriteLine("{0} samples from {1} to {2}", processor.NumSamples, startTime, endTime);
        Console.WriteLine("Processor time: {0}", processor);
        Console.WriteLine("Paging: {0}", paging);
        Console.WriteLine("Free bytes: {0}", freeBytes);
        Console.WriteLine("Query took {0}", queryTimer.Elapsed);
    }
}
```

The PersistentDictionary is multi-thread safe so it is fine to run (multiple) queries while the dictionary is being updated. It is possible to have multiple threads updating the dictionary as well.

## Putting it all together

Now we just need a main method.

```c#
/// <summary>
/// The main method. Called on program startup.
/// </summary>
/// <param name="args">Arguments to the program.</param>
public static void Main(string[]() args)
{
    const string DictionaryPath = "SystemStats";

    if (0 == args.Length)
    {
        CollectStats(DictionaryPath);
    }
    else if (2 == args.Length)
    {
        DumpStats(DictionaryPath, DateTime.Parse(args[0](0)), DateTime.Parse(args[1](1)));
    }
    else
    {
        if (!PersistentDictionaryFile.Exists(DictionaryPath))
        {
            Console.WriteLine("No stats collected");
        }
        else
        {
            using (PersistentDictionary<DateTime, Sample> dictionary = new PersistentDictionary<DateTime, Sample>(DictionaryPath))
            {
                // Getting the first item, last item or count of items are all O(1) operations.
                Console.WriteLine(
                    "Stats available from {0} to {1} ({2}) records",
                    dictionary.Keys.FirstOrDefault(),
                    dictionary.Keys.LastOrDefault(),
                    dictionary.Count);
            }                    
        }
    }
}
```

## Performance

These numbers are from a database where samples were taken 100 times/second, there are slightly over 500,000 entries and the database is around 1.5Gb.

```
Stats available from 2/15/2011 12:59:48 PM to 2/16/2011 11:30:15 AM (7560591) records
```

Here are the results of a few queries. Each run is a new process so none of the database is cached. In a real-world scenario a process might keep the dictionary open so subsequent queries would be even faster:

```
>.\SystemStats.exe "2/15/2011 4:34:30 PM" "2/15/2011 4:34:33 PM"
235 samples from 2/15/2011 4:34:30 PM to 2/15/2011 4:34:33 PM
Processor time: min = 0.00, max = 100.00, average = 52.47
Paging: min = 0.00, max = 872.37, average = 4.11
Free bytes: min = 3,659,247,616.00, max = 3,678,650,368.00, average = 3,667,921,270.74
Query took 00:00:00.1053856

>.\SystemStats.exe "2/15/2011 2:13:00 PM" "2/15/2011 2:15:00 PM"
11805 samples from 2/15/2011 2:13:00 PM to 2/15/2011 2:15:00 PM
Processor time: min = 0.00, max = 100.00, average = 38.04
Paging: min = 0.00, max = 4,100.28, average = 3.39
Free bytes: min = 4,026,720,256.00, max = 4,069,093,376.00, average = 4,063,535,199.76
Query took 00:00:00.8090867

>.\SystemStats.exe "2/15/2011 5:23:00 PM" "2/15/2011 6:25:00 PM"
346751 samples from 2/15/2011 5:23:00 PM to 2/15/2011 6:25:00 PM
Processor time: min = 0.00, max = 100.00, average = 37.36
Paging: min = 0.00, max = 36,016.09, average = 11.27
Free bytes: min = 3,183,525,888.00, max = 3,589,419,008.00, average = 3,413,498,109.31
Query took 00:00:21.6863322
```

The query performance is proportional to the number of entries processed, it will be basically independent of the database size.
