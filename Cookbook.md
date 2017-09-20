## How do I create a new sequence?

To create a genomic sequence, you use the `Bio.Sequence` class. These are read-only, in-memory arrays of data which an associated alphabet.

```C#
Sequence sequence = new Sequence(Alphabets.DNA, "AGCGTCCATG");
```

There are different sequence implementations available, and the sequence concept is abstracted through an interface `ISequence` which should be used when passing sequences around.


## How do I select an alphabet

There are two ways to get alphabets: directly through the corresponding class, or through the Alphabets static class.

.NET Bio supports several alphabet styles oriented around DNA, RNA and Proteins. For each group, there are two sub-groups: _base_ and _ambiguous_.

The `Alphabets` class has properties which let you select an alphabet, and methods to identify an alphabet based on a set of characters, or to perform validations on alphabets (to check if a given alphabet is included in another for example).

For example, to use the base DNA alphabet, you can use:

```C#
var dnaAlphabet = Alphabets.DNA;
```

to use the ambiguous form, it would be:

```C#
var dnaAlphabet = Alphabets.AmbiguousDNA;
```

As mentioned, you can also go directly to the given class and use the static `Instance` property; for example:

```C#
var dnaAlphabet = DnaAlphabet.Instance;
```

Once you have an alphabet, you can examine the contents (valid symbols) or use it to construct a sequence.


## How do I load sequences from a file?

To load sequences from a file-based format you need a _parser_. Given a filename you can locate the parser associated with it pretty easily with code like this:

```C#
// Find the parser
const string Filename = @"4_raw.fasta";
ISequenceParser parser = SequenceParsers.FindParserByFileName(Filename);
if (parser == null) {
   Console.WriteLine("Unable to locate parser for {0}", Filename);
   return;
}
```

You can also deliberately create the parser if you know the exact type you want:

```C#
ISequenceParser parser = new Bio.IO.FastA.FastAParser(); 
```

Once you have the parser, you can then either pass it a Stream from your given platform, or use the extension methods to work with files or `IStorageFile` on Windows. Here's an example of opening a file and parsing out all the sequences and capturing them in a list:

```C#
// Parse the file.
List<ISequence> sequences;
using (parser.Open(Filename)) 
{
   sequences = parser.Parse().ToList();
}
```

Notice the use of the `using` block, this will ensure the file is closed as soon as the data is read, parsed and turned into the list. You should follow this paradigm when you are working with files to conserve resources.


## How do I save sequences to a file?

To save a set of sequences to a file, you need a _formatter_. You can identify the available formatters for sequences using the SequenceFormatters class by either filename or format name:

```C#
const string Filename = "output.fa";
ISequenceFormatter formatter = SequenceFormatters.FindFormatterByFileName(Filename);
```

If it returns null, then the formatter could not be identified by the filename. You can also create a formatter directly if you know exactly what format you want to use, for example to use Fasta:

```C#
ISequenceFormatter formatter = new Bio.IO.FastA.FastAFormatter();
```

Once you have the formatter and the sequences you can persist either to a Stream by calling the Write method, or use the platform-specific extension methods to work with the file APIs available for your platform, for example to work with a desktop app:

```C#
bool WriteToFile(string filename, IEnumerable<ISequence> sequences)
{ 
   ISequenceFormatter formatter = SequenceFormatters.FindFormatterByFileName(filename);
   if (formatter == null) return false;
   using (formatter.Open(filename)) {
      formatter.Write(sequences);
}
```

## How do I issue a BLAST request to NCBI?

To execute a BLAST search request to NCBI, you use an `IBlastWebHandler`. This recipe shows how to use the `NcbiWebHandler` class, but you can pick a different BLAST service or even call your own with the same basic steps.

Assuming you have a sequence, or a set of sequences you want to look for you can execute the following code:

```C#
async List<BlastResult> DoBlast(IEnumerable<ISequence> sequences)
{
   // Setup the Blast Request parameters. Must select the database and program.
   // Can have other parameters if necessary (such as email).
   BlastRequestParameters bp = new BlastRequestParameters(sequences) {
	Database = "nt",
	Program = BlastProgram.Megablast,
   };

   // Create the web handler.
   IBlastWebHandler webHandler = new NcbiBlastWebHandler();
   
   // Execute the request. Note the use of async / await here.
   // This returns a Stream with the XML data from NCBI.
   var response = await webHandler.ExecuteAsync(bp, CancellationToken.None).Result;

   // Parse the results to an object graph. Can also consume the results directly
   // it comes back in XML form.
   IBlastParser blastParser = new BlastXmlParser();
   return blastParser.Parse(response).ToList();
```

The returning data will include the hits and high-scoring pairs.


## How do I align two sequences?

To align two sequences, pick the aligner you'd like to use, set it up with parameters and then call the `Align` method, passing your sequences.

```C#
using Bio;
using Bio.Algorithms.Alignment;
using Bio.SimilarityMatrices;
...

// Create two sequences; normally you'd have this already.
ISequence dna1 = new Sequence(Alphabets.AmbiguousDNA, "ACTGAAGGATATTA");
ISequence dna2 = new Sequence(Alphabets.AmbiguousDNA, "ACTGTCCTAGATATTA");

// Pick our aligner; there are several to choose from; all in the 
// Bio.Algorithms.Alignment namespace.
var algo = new Bio.Algorithms.Alignment.NeedlemanWunschAligner(); 

// Setup the aligner with appropriate parameters
algo.SimilarityMatrix = new SimilarityMatrix(
                     SimilarityMatrix.StandardSimilarityMatrix.AmbiguousDna);
algo.GapOpenCost = -6;
algo.GapExtensionCost = -1;

// Execute the alignment.
var results = algo.Align(dna1, dna2);

// Display / Process the results.
Console.WriteLine("Pairwise Alignment: " + results.Count + " result entries");
foreach (IPairwiseSequenceAlignment ipsa in results) 
{
   // Note equiv: ipsa.ipsa.PairwiseAlignedSequences[0].ToString()
   Console.Write(ipsa.ToString());
   Console.WriteLine("Processing Pairwise Alignments: " 
                                  + ipsa.PairwiseAlignedSequences.Count 
                                  + " entries");
   foreach (PairwiseAlignedSequence pas in ipsa.PairwiseAlignedSequences)
      Console.Write("Alignment Score:" + pas.Metadata["score"].ToString());
}
```


## How can I search a sequence for a pattern efficiently?

There is a [Boyer-Moore search algorithm](http://www-igm.univ-mlv.fr/~lecroq/string/node14.html) in .NET Bio, as well as support for [Suffix Trees](http://en.wikipedia.org/wiki/Suffix_tree) which are good for this sort of thing. 

They are both located in the `Bio.Algorithms.StringSearch` namespace. You could use it like this:

```C#
ISequence sequence = new Sequence(Alphabets.AmbiguousDNA, "AGCTAGGTAGCTCAAAAAAGGG"); 
IPatternFinder searcher = new BoyerMoore()
{ 
   IgnoreCase = true 
};

foreach (var pos in searcher.FindMatch(sequence, "*GCTCA*GGG"))
{
   Console.WriteLine("Found Match at: {0}", pos);
}
```

## How do I download a sequence directly from NCBI?

To download a GenBank-formatted genome from the NCBI FTP server, you need to know the name of the folder in which the file is stored and the accession code for the genome. In this example we download a bacterial genome from the folder genomes/Bacteria.

First create a string with the address of the file and a `WebClient` to download content

```C# 
const string bacteriaTemplate = "ftp://ftp.ncbi.nlm.nih.gov/genomes/Bacteria/{0}/{1}.gbk";
const string accession = "NC_002182";
const string folder = "Chlamydia_muridarum_Nigg_uid57785";
string url = String.Format( bacteriaTemplate, folder, accession );

WebClient downloader = new WebClient();
```

Once you have the `WebClient` you can create a `Stream` from which the sequence can be parsed. 

```
using ( Stream stream = downloader.OpenRead( url ) )
{
	StreamReader reader = new StreamReader( stream );
	ISequenceParser parser = new GenBankParser();
	ISequence sequence = parser.Parse( reader ).FirstOrDefault();

	// Process sequence here...
}
```

**Notes:**
- If the file is very large or you are developing an interactive application, you can use the `WebClient.OpenReadAsync` method to do a non-blocking download.

- The parser can be used in several ways to access data. In this example, we use a `StreamReader` to access the downloaded data stream.

- A file may contain more than one sequence, so the `Parse` operation produces a list of sequences. In this case, we know there is only sequence, so we use `FirstOrDefault()` to grab the first element of the list.


## How do I use GenBank sequence metadata?

Unlike other sequence file formats such as FASTA which are primarily for communicating sequence data, the GenBank (**.gbk**) document format supports detailed annotation of features such as genes. 

The sequence produced by a GenBank parser is an object which contains, along with raw sequence data, the full collection of feature annotations. This example shows how to access the annotations. 

First, load a sequence from a GenBank file:

```C#
const string genbankFileName = @"...";
ISequence sequence = null;

using ( ISequenceParser parser = new GenBankParser( genbankFileName ) )
{
	IEnumerable<ISequence> sequences = parser.Parse();
	sequence = parser.Parse().FirstOrDefault();
}
```

Some basic information about the genome can be obtained directly from the ISequence object, but feature annotations are stored in a metadata object within the `ISequence`. To get a reference to the GenBank metadata:

```C#
GenBankMetadata metadata = sequence.Metadata["GenBank"] as GenBankMetadata;
```

The metadata object has several properties that provide more detail about the genome:

```C#
// Get general information about the sequence.
Console.WriteLine( "Locus: {0}", metadata.Locus );
Console.WriteLine( "Accession: {0}", metadata.Accession );
Console.WriteLine( "Version: {0}", metadata.Version );
Console.WriteLine( "Definition: {0}", metadata.Definition );
Console.WriteLine( "Common name: {0}", metadata.Source.CommonName );
Console.WriteLine( "Species: {0}", metadata.Source.Organism.Species );
Console.WriteLine( "Genus: {0}", metadata.Source.Organism.Genus );
Console.WriteLine( "Class levels: {0}", metadata.Source.Organism.ClassLevels );
```

To examine all annotated features in a genome:

```C#
// List all features in the collection.
foreach ( FeatureItem feature in metadata.Features.All )
{
	Console.WriteLine( "Feature: {0}", feature.Key );
	Console.WriteLine( "\tLocation: {0}", feature.Location );
	Console.WriteLine( "\tSubsequence: {0}", feature.GetSubSequence( sequence ) );
	Console.WriteLine( "\tQualifiers:" );

	foreach ( KeyValuePair<string,List<string>> qual in feature.Qualifiers )
	{
		foreach ( string value in qual.Value )
		{
			Console.WriteLine( "\t\t{0}: {1}", qual.Key, value );
		}
	}
}
```


## How do I generate synthetic sequence data?

Synthetic sequences can be generated by using the read simulator tool which ships as part of the download. The read simulator classes don’t form part of the .Net Bio API, but can be accessed via the overall source code drop. 

Classes needed are:

- ReadSimulator.cs
- SimulatorSettings.cs

Assuming you have an `ISequence` object loaded with data from NCBI.

```C#
ISequence sequence; 
var simulation = new ReadSimulator(); 

//set the simulation settings 
// the Simulation settings class can be configured to use three different default settings as outlined 
//in the notes below. 
// the ReadSimulator constructor will need to be altered to take in the DefaultSettings object as a parameter to allow for this change 

SimulatorSettings Settings = new SimulatorSettings(); 
DefaultSettings defaultSettings = DefaultSettings.SangerDideoxy; Settings.SetDefaults(defaultSettings); 

//set sequence to use in simulation 
simulation.SequenceToSplit = sequence; 

//perform simulation 
//File is created in the project bin/debug folder containing the synthetic sequence data 
simulation.DoSimulation("outputFile"); 
```

Example output is given below. Here we shown only the first ‘read’ in its entirety, and the first line or so of the second ‘read’. This example generates 100 reads. A second example output is given below using the ShortRead settings from the Simulator Settings class, which restricts the output. 

```
NC_002182 (Split 1, 1061bp)
atgtgtgcatagtagatactccacctagtcttggtggattaacaaaagaagcctttattgcaggagacaaactaatcgta
tgtttgattcctgagccattttctattctcgggctgcagaaaattagagaatttttaatttctataggcaaacctgagga
agaacatattcttggggtagcactatctttttgggatgaccggagttcgactaatcaaacgtacatagatatcattgagt
caatttacgaaaataagattttttcaacaaaaatacgcagagatatttctttgagtcgttcccttcttaaagaggattct
gtgatcaatgtatatccaacttcaagagctgcaacagatattctgaatttaacacacgaaatatctgctcttttaaattc
taaacacaaacaagacttttcccagaggacactgtgaataaactggaaaaggaagctagcgtcttttttaaaaaaaatca
ggaatccgtttctcaagactttaagaaaaaggtttcttcaattgagatgttttcaacttctttaaattcggaggaaaacc
agagtctggatcggctttttttgtctgagactcagaatttatcagatgaagaatcttaccaagaagatgttttgtcagta
aaacttctgacaagtcaaataaaggctattcaaaaacaacacgtgctccttcttggagagaagatttacaatgcgagaaa
gatactaagtaaaagttgtttctcttcaacaaccttttcatcttggctagatttagttttcaggactaaatcatccgcct
ataatgcgttggcttattatgaacttttcataagtctaccaagcacaactttgcagaaagagttccaatcaatcccgtat
aagtctgcatatattttagctgctaggaaaggagacttaaaaacaaaagtctctgttatagggaaagtttgtggaatgtc
caatgcatctgctatccgggttatggaccaacttcttccttcatctagaagtaaagataatcaaagatttttcgaatctg
atttagagaaaaatcgacagt

>NC_002182 (Split 2, 1053bp)
caaaacatctagaaacaaatacgaatttagtgggaaagaatctgaaacagctttagaggctctgtatcatttaggacatc
....
```

Output using ShortRead settings, showing the first two reads from the output file. These settings generated 20272 reads in different, sequential output files

```
NC_002182 (Split 1001, 41bp)
acctcttacagatcaacaaataatacttgggacatcgacaa
NC_002182 (Split 1002, 32bp)
agccttgcgtatattttaaggatgaatcgata
```

**Note:**
The above example code excludes parameters from the ReadSimulator class constructor that pertain to a GUI update.

The ReadSimulator class constructor (and SimulationSettings class variable) needs to be altered as shown below.

```C#
public SimulatorSettings Settings; 

public ReadSimulator(SimulatorSettings Settings)
{
   _seqRandom = new Random();
   this.Settings = Settings;
}
```

The `SimulatorSettings` class has three default setting as follows. Each offers differences on depth of coverage, sequence length, length variation, error frequency, and distribution type:

- PyroSequencing
- SangerDideoxy
- ShortRead
