
As a software developer I think we've all had to deal with a CSV file at some point or another. I was recently analyzing a [large-ish CSV data set](https://github.com/CSSEGISandData/COVID-19/blob/master/csse_covid_19_data/csse_covid_19_time_series/time_series_covid19_confirmed_US.csv) and used my go-to library, [Lumenworks](https://www.codeproject.com/Articles/9258/A-Fast-CSV-Reader), to do the heavy lifting. 

Lumenworks made short work of what I wanted to get done, but it reminded me of an optimization I had been thinking about recently for parsing structured text data. This lead me down a rabbit hole of implementing my own CSV parser: (Sylvan.Data.Csv)[https://www.nuget.org/packages/Sylvan.Data.Csv/], and yeah, I know what you're thinking, "just what the world needs, another CSV parser", and you're right. Regardless, I was curious if my optimization would provide any improvement in performance over the current standards.

## The Naive, Broken CSV parser

The first implementation that every implements is `TextReader.ReadLine`, followed by a `String.Split`. This is actually decent implementation, assuming the CSV input meets certain constraints. The problem is that CSV, as specified by [RFC 4180](https://en.wikipedia.org/wiki/Comma-separated_values), is a bit more complex the naive approach can handle. As soon as a quoted field containing a delimiter is introduced, things will breakdown and a true parser will be needed. 

Perhaps you control the data source, and know that the data is constrained such that it will work with the naive approach. But, more likely than not, you'll need something more robust.

With [literally hundreds](https://www.nuget.org/packages?q=tag%3Acsv) of options for CSV parsers available on Nuget, it might be overwhelming to select one for your needs. Given that I'm adding one to the field, I decided to do an analysis of what was currently available.

## Lumenworks.Framework.IO

I discovered Sébastien Lorion's CSV reader library through the [codeproject aricle](https://www.codeproject.com/Articles/9258/A-Fast-CSV-Reader) well over a decade ago. Even back then, I was impressed with the performance as well as the error handling capabilities his library provides. It remains an excellent option, though the code now appears to be maintained by [Paul Hatcher](https://github.com/phatcher/CsvReader).

The library is well designed, and intuitive to use. The primary class `CsvReader` implements the BCL `IDataReader` interface, which makes it convenient to use for common data interchange scenarios. The library includes a lot of additional functionality, such as caching, that I haven't had need of. It remains a competitive option in 2020.

## CsvHelper

Josh Close's [CsvHelper](https://github.com/JoshClose/CsvHelper) appears to be the de-facto CSV parsing library for dotNET in 2020. It has the most downloads (21m) of all csv related nuget packages. **This is probably the library you'll want to use.** It is [very well documented](https://joshclose.github.io/CsvHelper/), and appears to be very actively supported and enhanced.

CsvHelper is very easy to use; the primary class implements IDataReader. The library also includes convenience extensions for binding the CSV data directly to objects. The raw performance using `CsvDataReader` is  competitive, but slightly slower than Lumenworks.

## NLight CSV

While Sébastien Lorion no longer supports his Lumenworks parser, he has a replacement library [NLight]() with a new CSV reader that is just as capable, and more performant, than Lumenworks. This library contains a lot of other, unrelated, code that may or may not be useful. This library doesn't have a lot of nuget downloads, so it is easy to overlook, but it is an excellent library.

## VisualBasic

The Microsoft.VisualBasic includes a CSV parser class. I metion this, because it is owned by Microsoft, and is part of the BCL. It gets the job done, but of all the options I benchmarked it was the slowest, and consumed the most memory. The API also felt a little weird to me, but that might be because of it's VB heritage. Not to be critical of VB, but the shape of the API doesn't feel idiomatic in C#.

## The Horrorshow

There are a handful of excellent choices for parsing CSV files, but unfortunatly, there are also a lot of really bad options. I investigated myriad csv packages on nuget, and more often than not I immediately disregarded them. There are, unfortunatley, too many to bother listing but I can describe the common problems, and my goal isn't to shame.

The most common problem was an immediately broken API. If the primary constructor for the CSV parser only accepts a `Stream`, and *doesn't* also accept an encoding, that's a non-starter. There were a few libraries like this. However, there were also libraries that accepted a `Stream` and `Encoding` but didn't offer a `TextReader` option. This is also limiting and inconvenient for some scenarios. Likewise, many packages only accept a *file name*. No thanks.

I also investigated implementations that used the naive approach, or regex in ways that made me immediately abandon them. The naive approach is 4 lines of code; why put that in a library?

A couple examples provided no way to consume the CSV data without binding it to a strongly-typed object. My use-case data set didn't lend itself to this approach, so I abandoned these options. Some of them might be excellent, but the CsvHelper library provides this capability while also allowing to consume the data dynamically.

Some examples I couldn't even figure out how to consume the library. They felt over-engineered for what they were.

## NReco

I eventually came across an implementation that really impressed me performance-wise: [NReco](https://github.com/nreco/csv). This library is *fast*. This was the first library I came across that was on par with the performance of my own library; in fact, it marginally beat it. I was curious how the authors achieved that, so I investigated the details of the implementation. Somewhat surprisingly, the NReco author uses the exact same strategy that I'd used in my own CSV parser, but they had managed to micro-optimize a bit better.

This is a tiny library that only includes a raw reader and writer for CSV files. The CsvReader doesn't implement any BCL base class or interface, so it would need to be adapted for certain uses. The CsvReader is also somewhat limited in what options it supports regarding quotes, escapes, headers, etc. Specifically, it *doesn't* support a header row directly, and the consumer must deal with this themselves. Not that big an issue, but it could be a bit more convenient.

The implementations appears to be solid, the performance is fantastic, but the API is perhaps a bit lacking.

## Sylvan

Finally, I throw my own library into the mix. After grinding down a few hot-spots it is now the fastest, and lowest memory usage, of the CSV parsing libraries that I've benchmarked. It is *not* as feature rich as a lot of the other options, but it does provide a good set of options. First, it derives from `DbDataReader` and thus `IDataReader` so it provides a familiar API surface. It supports async reading, which none of the other libraries I tested support.

The biggest limitation, is that I've set a limit on the size of record that can be read. This is required by the





|        Method |       Mean |     Error |    StdDev |     Median | Ratio | RatioSD |      Gen 0 |    Gen 1 |   Gen 2 |    Allocated |
|-------------- |-----------:|----------:|----------:|-----------:|------:|--------:|-----------:|---------:|--------:|-------------:|
|    Lumenworks |  16.902 ms | 0.2369 ms | 0.2100 ms |  16.874 ms |  1.00 |    0.00 | 10093.7500 |        - |       - |  41261.54 KB |
|   NaiveBroken |   4.611 ms | 0.0203 ms | 0.0189 ms |   4.608 ms |  0.27 |    0.00 |  2664.0625 |        - |       - |  10883.46 KB |
|     CsvHelper |  22.016 ms | 0.0704 ms | 0.0658 ms |  22.026 ms |  1.30 |    0.01 |  6437.5000 |        - |       - |   26308.7 KB |
|     NLightCsv |  12.463 ms | 0.0834 ms | 0.0739 ms |  12.444 ms |  0.74 |    0.01 |  1703.1250 |  15.6250 | 15.6250 |    7066.5 KB |
| FastCsvParser |  10.017 ms | 0.0469 ms | 0.0439 ms |  10.003 ms |  0.59 |    0.01 |  1765.6250 | 109.3750 | 46.8750 |   7292.67 KB |
|   VisualBasic | 107.147 ms | 2.3503 ms | 4.4144 ms | 105.623 ms |  6.43 |    0.39 | 44400.0000 |        - |       - | 182034.58 KB |
|     HansenCsv | 143.023 ms | 2.8187 ms | 3.4616 ms | 143.028 ms |  8.50 |    0.24 | 20500.0000 | 250.0000 |       - |  84001.02 KB |
|         NReco |   6.787 ms | 0.1337 ms | 0.2478 ms |   6.648 ms |  0.41 |    0.02 |  1710.9375 |   7.8125 |       - |   6993.29 KB |
|        Sylvan |   6.092 ms | 0.1402 ms | 0.1558 ms |   6.083 ms |  0.36 |    0.01 |  1710.9375 |  46.8750 | 23.4375 |   7045.27 KB |
|   NRecoSelect |   2.703 ms | 0.0468 ms | 0.0438 ms |   2.690 ms |  0.16 |    0.00 |   136.7188 |  35.1563 | 35.1563 |    567.15 KB |
|  SylvanSelect |   2.576 ms | 0.0392 ms | 0.0367 ms |   2.556 ms |  0.15 |    0.00 |    85.9375 |  31.2500 | 31.2500 |    355.55 KB |



