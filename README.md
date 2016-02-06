# librsfs
C++ library for the RuneScape file store

# Filesystem structure
The RuneScape file system consists of a three-level hierarchy, which can be illustrated as follows:

 - Filesystem
   - Index
     - Folder
         - File

In this hierarchy, the `filesystem` is simply put the directory containing the `main_file_cache.dat2` file 
(which contains the actual file data). The file system also has files in that directory ending with `.idx*` which 
ranges from `.idx0` (which would be a simple `index` with ID 0) to `.idx255` (which is a special one; index 
index 255 contains the root reference table). These indices contain `folder` entries. These folder entries can 
either be plain (compressed) data themselves, or contain individual files. 

# Building

## On Windows

### librsfs
Since it's pretty much impossible to let CMake find everything needed immediately on Windows you'll most likely have to do some -D'ing yourself when invoking the CMake command.

First of all, make a new directory. Out-of-source builds are so much nicer (and you don't get all the Git clobbing!).

```
mkdir librsfs-build
cd librsfs-build
```

Now invoke the CMake command with the defines to ensure everything's easy to find:

```
cmake ../librsfs -G "Visual Studio 12 Win64"
```

The above command builds for an x64 release. If you are looking for the 32-bit build, leave out Win64 when passing the generator to CMake. Note that this requires you to have built both ZLib and BZip2 yourself in advance, otherwise building will fail because there are no libraries.

This will create a solution file, a few project files and the other good stuff for you ready to open & build.

## On any other programmer-friendly operating system

It's less hassle on a unix system (by far) as CMake can properly detect everything you need, and what's missing can be installed through your package manager. Even it that fails, it's probably a two second job to build the libraries from source. CMake will simply generate a makefile which is ready to be invoked.


# Usage

## Loading from a folder

```c++
try {
	FileSystem filesystem;
	filesystem.LoadIndex(your_directory);
} catch (const std::exception &exc) {
	printf("Error loading directory: %s\n", exc.what());
}
```

## Checking the number of indices

```c++
auto count = filesystem.GetIndexCount();
```

## Checking the (assumed) number of entries in an index

This does not necessarily check the actual number of 'valid' entries, but guesses the number based on the highest entry ID. Since the entries could look like `1, 4, 77, 78, 79` this does not accurately match the actual number of entries containing data. The method would, using above data, return 79, while it actually only has 5 entries.
```c++
auto highest_entry_id = filesystem.GetIndex(0).GetEntryCount();
```

## Getting information on an entry

To acquire information about an entry (its starting offset and its size), use the method shown below.
```c++
auto model_folder_1234 = filesystem.GetIndex(7).GetFolderInfo(1234);
auto size = model_folder_0.GetSize(); // Full size of this entry (in compressed form, that is)
auto offset = model_folder_0.GetOffset(); // Absolute file offset where the data starts
```

## Reading data from an entry

Reading data is easy to do. Note that this data may be compressed!

```c++
vector<char> entry_data;
// see above on how to get model_folder_1234 defined
auto num_read = filesystem.Read(model_folder_1234, entry_data);
```

## Decompressing an entry

Some entries are compressed. Actually, most of them are. They are either compressed with gzip or with bzip2, or LZMA in the newest RS3. Librsfs supports gzip, bzip2 and, when found on the build path, lzma (which is used in the later versions of RuneScape 3).

```c++
vector<char> decompressed;
int num = Compression::Decompress(entry_data, decompressed);
```

## Getting compression information

If you're simply curious on the compression type or the inflated/deflated sizes, you can do that with:
```c++
CompressionInfo compression_info(entry_data);
auto compression_type = compression_info.GetCompressionType();
auto decompressed_size = compression_info.GetDecompressedSize();
```
