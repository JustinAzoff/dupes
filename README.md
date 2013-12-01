dupes
=====

Find duplicate files.

## finding duplicate files

The program is a little bare, and needs a nicer API, but the method it uses is
the most efficient one that I am aware of.

There are a couple of different ways you can find duplicate files:

### Compute the hash of all the files, and look for duplicates
This method works well if the files on disk are mostly static, and files are
added infrequently.  In this case you can compute the hashes once, and keep it
around for later scans.  However, if you are only running the scan once, this
method is not ideal since it requires you to read the full contents of every
file 

### Compute the hash of files with the same size
This is the method that I think fdupes still uses. It first builds a candidate
list of files that are the same size, and computes the checksum of each.  This
method works well if most of the files that are the same size are really
duplicates, but otherwise triggers too much unneeded IO.

### Compare all files with the same size in parallel
This is the method that my program uses.  Like fdupes, I first built up a
candidate list of files with the same size. Instead of hashing the files,
it simply reads each file at the same time, comparing block by block.
This is just like what the *cmp(1)* program does, but for multiple files at the
same time.  The benefit of this over calculating the files hash, is that
as soon as the files differ, you can stop reading.

##Implementation
There are a couple of things you need to keep in mind to implement this method.

### Don't open too many files.
You have to be careful not to try and open too many files at once.  If the user
has 5,000 files that all have the same size, the program shouldn't try and open
all 5,000 at once.  My program uses a simple helper class to handle opening and
closing files.  The default blocksize in my program would probably waste a bit
of memory in this case, but that is easily changed.

### Correctly handle diverging sets.
Imagine the filesystem contains 4 files of the same size, 'a', 'b','c', and 'd',
where a==c, and b==d.  While reading through the files, it will become clear
that a!=b, a==c, and a!=d.  It is important that at this step the program
continues searching using (a,c) and (b,d) as possible duplicates.  This is
implemented using recursion, the sets (a,c) and (b,d) are fed back into the
duplicate finding function.


##Example run, compared to fdupes.
Here is dupes.py running against fdupes on a modestly sized directory.
Notice how dupes.py only needs to read 600K(not counting metadata).

According to iofileb.d from the dtrace toolkit, dupes.py reads 10M of data (which
I think includes python), and fdupes reads 517M.  This alone explains the 20x speedup
seen in dupes.py

    justin@pip:~$ du -hs $DIR
    15G   $DIR

    justin@pip:~$ time python code/dupes.py $DIR
    2896 total files
    35 size collisions, max of length 5
    bytes read 647168

    real    0m1.224s
    user    0m0.234s
    sys     0m0.494s

    justin@pip:~$ time fdupes -r $DIR
    real    0m41.694s
    user    0m13.612s
    sys     0m7.491s

    justin@pip:~$ time python code/dupes.py $DIR
    2896 total files
    35 size collisions, max of length 5
    bytes read 647168

    real    0m3.662s
    user    0m0.256s
    sys     0m0.568s

    justin@pip:~$ time fdupes -r $DIR
    real    0m55.473s
    user    0m11.383s
    sys     0m6.433s
