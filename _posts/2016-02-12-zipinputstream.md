---
layout: post
title: TIL how ZipInputStream works
---
Today, I was working on an `OutOfMemory` issue when uploading files to a Tomcat server. Specifically, if more than one medium-sized zip file (about 20 Mb uncompressed) was uploaded simultaneously, the server would immediately run out of memory and start sending out 500s.

The first part of diagnosing the problem was to see what was using all of that memory. For that, I used a tool called `jmap` to do a heap dump while uploading some files, and then I used [JVisualVM](https://docs.oracle.com/javase/8/docs/technotes/tools/windows/jvisualvm.html) to examine the results.

First, I had to trigger the error. To do that, I uploaded a bunch of files at once while monitoring the memory usage of the Tomcat process using JVisualVM to monitor memory usage:

{% highlight bash %}
python mass_upload.py 1 & python mass_upload.py 1 & \
python mass_upload.py 1 & python mass_upload.py 1 wait
{% endhighlight %}

`mass_upload.py` is just a script that uploads a zip file to the server. While this was running, I was able to see the large increase in memory usage in the **Monitor** tab of JVisulVM:

<div class="center"><img src="{{ site.url }}/images/memory.png" alt="Memory usage"></div>

Once I saw the memory usage shooting up, I ran `jmap` to create a heap dump, which will allow me to see what kind(s) of objects are using up all of that memory:

{% highlight bash %}
$> sudo jmap -F -dump:format=b,file=~/snapshot.jmap 5797
{% endhighlight %}

Once that's done, I now have a 700+ Mb file called `snapshot.jmap` in my home directory. I can import it into JVisualVM to examine it. By ordering the **Classes** tab by size, you can see what's taking up most of the memory:

<div class="center"><img src="{{ site.url }}/images/instances.png" alt="Byte arrays"></div>

What are all of these `byte[]` objects doing? It turns out that they are the (unzipped) files being loaded into memory. As the zip files were being uploaded, we were unzipping them and loading each one fully into memory. When a bunch of files were being uploaded at the same time, the combined size of those files was causing the server to run out of memory.

Here was the offending piece of code:

{% highlight java %}
int i;		
byte[] buff = new byte[1024];		
ByteArrayOutputStream byteOut = new ByteArrayOutputStream();		
while ((i = this.read(buff, 0, buff.length)) > 0) {		
  byteOut.write(buff, 0, i);		
}		
	
return new ByteArrayInputStream(byteOut.toByteArray());
{% endhighlight %}

Where `this` is a wrapper around a Java [ZipInputStream](https://docs.oracle.com/javase/7/docs/api/java/util/zip/ZipInputStream.html). This is pretty wasteful -- we can do better. Instead of reading the entire file into memory, we can just delete the code and use the `ZipInputStream` directly.

The question remains, though: why were we doing it like this in the first place? I think that it was due to a misunderstanding by the original writer of how a `ZipInputStream` actually works when you have multiple files -- a misunderstanding that took me the better part of the day to figure out. Here's what I was able to discover:

* A `ZipInputStream` is composed of one or more files, each of which has an associated `ZipEntry` object. The `ZipEntry` object contains metadata about the zipped file (the name of the file, the compressed and uncompressed size of the file, etc).
* To read the data for the first entry from the stream, you have to call `getNextEntry()` before doing any reading. `getNextEntry()` returns that file's `ZipEntry` object and positions the stream at the beginning of that file. Subsequent calls to `read()` will read data for that `ZipEntry`'s file.
* When `read()` starts returning `-1` (i.e. there's no more data), that **does not** mean that you've read the entire zip file. It only means that you're done with that particular `ZipEntry`. Calling `getNextEntry()` again will take you to the next file in the archive, where you can repeat the process over again.
* Only when `getNextEntry()` returns `null` are you actually finished with everything in the zip file.

Once I was able to figure this out -- through a combination of Googling, reading documentation, and experimenting in a toy program -- the solution was straightforward. Instead of reading each file into a `ByteArrayInputStream` and passing that around, we can just use the `ZipInputStream` directly and call `getNextEntry()` every time we want to read the next archived file.
