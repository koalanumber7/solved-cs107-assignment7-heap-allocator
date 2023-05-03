Download Link: https://assignmentchef.com/product/solved-cs107-assignment7-heap-allocator
<br>
In completing this assignment, you will:

appreciate the complexity and tradeo s in implementing a heap allocator further level up your pointer-wrangling and debugging skills pro le your code to nd ine        ciencies and work to streamline them consider improvements against the complexity of code required to achieve them bring together all of your CS107 skills and knowledge into a satisfying capstone experience– you can do it!

<h1>Overview</h1>

Watch video walkthrough! (https://www.youtube.com/watch?v=3Ix8A1rOy9w)

You’ve been using the heap all quarter, now is your chance to peel back the covers and work out the magic underpinnings of malloc , realloc , and free for yourself! The key goals for a heap allocator are:

correctness: services any combination of well-formed requests tight utilization: compact use of memory, low overhead, recycles high throughput: can service requests quickly

There are a wide variety of designs that can meet these goals, with various tradeo s to consider. Correctness is non-negotiable, but is the next priority on conserving space or time? A bump allocator can be crazy-fast but chews through memory with no remorse. Alternatively, an allocator might pursue aggressive recycling and tight packing to squeeze into a small memory footprint but execute a lot of instructions to achieve it. A industrial-strength production allocator aims to nd a sweet spot without sacri cing one goal for the other. This assignment is an exploration of implementing a heap allocator and balancing those tradeo s.

<h1>Getting started</h1>

Check out a copy of your cs107 repository with the command:     git clone /afs/ir/class/cs107/repos/assign7/$USER assign7

<h1>1)   Code study: test harness</h1>

In order to put a heap allocator through its paces, you will need a good testing regimen. Our provided test_harness.c will be a big help! It allows you to test an allocator on inputs supplied as “allocator scripts”. An allocator script le contains a sequence of requests in a simple textbased format. The three request types are a (allocate) r (reallocate) and f (free). Each request has an id-number that can be referred to in a subsequent realloc or free.

A script le containing:

is converted into these calls to your allocator:

The test_harness program will parse a script le and translate it into requests. During execution, it checks that each request appears to be correctly satis ed and watches to con rm that the client’s payload data is properly maintained.

Read over test_harness.c so that you understand how it can help you test your allocator. It’s ne to skim over the script parsing, but please pay careful attention to the portions used to evaluate the allocator correctness. Answer these questions in your readme.txt le:

<ol>

 <li>Identify the required alignment for memory blocks and explain how it is checked by the test harness.</li>

 <li>How does the test harness verify that a memory block does not overlap any other?</li>

 <li>At what point does the test harness call validate_heap to con rm that the heap internals are valid?</li>

 <li>What value does the test harness write to the payload bytes? For what purpose is it writing</li>

</ol>

to the payload?

One easy-to-overlook detail is that the test_harness calls the allocator’s myinit function after executing one script and before starting another. The function is expected to wipe the slate clean and start fresh; if myinit doesn’t do its job, you may encounter bugs caused by ghost heap housekeeping leftover from the previous script. Keep this in mind for when implementing

myinit in your heap allocator!

<h1>2)   Code study: bump allocator</h1>

Your next task is to study the bump allocator. This code demonstrates one of the simplest possible approaches, with blindingly-high throughput and zero concern for utilization. You will be working toward a much more balanced implementation, but read over this code to be introduced to the basics of how a heap allocator might work. Answer these questions in your readme.txt le:

<ol>

 <li>The arguments to myinit are the bounds of the heap segment. Run test_bump under gdb and observe the boundaries. At what address does the segment start? How far does it extend? (hint: the segment size is not the same for all scripts, read test_harness.c to see how size is determined for a particular script) What happens if the allocator attempts to read/write past the end of the segment?</li>

 <li>Each memory block is required to be aligned on a 8-byte boundary. How does the bump allocator organize blocks so as to conform to this requirement?</li>

 <li>Free is implemented as no-op. Why is the bump allocator incapable of recycling memory?</li>

 <li>Its realloc operation makes no attempt to resize in place, it always goes with</li>

</ol>

malloc/memcpy/free. Focus in on the memcpy call that is used to move the payload data. Is it correct to use memcpy here or would memmove be more appropriate? Explain. How many bytes are copied from the old location to the new? This choice of size is a bit shy. It is clearly wasteful if newsz is signi cantly larger than the original payload size but it won’t cause an error to copy the excess data. Explain why it is guaranteed to succeed. (i.e.

correctly preserves contents of original payload and no error comes from reading/writing the excess bytes).

As a consequence of its wanton use of memory, the bump allocator fails on many scripts. Run the test harness on the sample scripts to identify which scripts it can and can’t process:

Once you know which scripts it can handle, run the bump allocator under callgrind to get some rough performance data. (See the advice page (advice.html) for tips on using callgrind to count instructions within the allocator functions.) Use the annotated source to answer the following question in your readme.txt le:

<ol>

 <li>Use the callgrind-annotated source to nd “hot spots”, i.e. those lines with signi cantly</li>

</ol>

higher concentration of instruction counts. Which lines are the hot spots of bump.c?

<h1>3)   Implement implicit free list</h1>

Now it’s your turn! The le implicit.c is ready and waiting for you to implement your own recycling allocator that puts the naive bump allocator’s memory-chomping approach to shame.

The speci c features that your implicit free list allocator must support:

Headers track block information (size, status in-use or free)

Free blocks are recycled and reused for subsequent malloc requests

Malloc searches heap for free blocks via an implicit list (i.e. traverses block-by-block) Realloc resizes block in-place whenever possible, e.g. if resize smaller or neighboring block(s) free and can be absorbed (this is a limited form of coalesce only for realloc, no expectation to support coalesce more broadly for implicit)

This allocator won’t be that zippy of a performer (an implicit free list is poky at best) but will do a good job of recycling freed nodes which enables it to successfully handle many workloads that cannot be accommodated by the bump allocator. Without coalescing, it is missing out on some opportunities to more aggressively recycle memory, but that comes up next in the explicit version!

The bulleted list above indicates the minimum speci cation that you are required to implement. Further details are intentionally unspeci ed as we leave these design decisions up to you; this means it is your choice how to structure your header, whether you search using rst- t versus next- t, and so on. It can be fun to play around with these decisions to see the various e ects, but any reasonable and justi ed choices will be acceptable for grading purposes.

Answer the following questions in your readme.txt:

<ol>

 <li>What lines of implicit.c are its “hot spots”?</li>

 <li>Compare the behavior/performance of your implicit free list allocator on the sample scripts to that of the bump allocator and brie y summarize your ndings. Your answer should consider di erences in robustness, throughput, and utilization.</li>

</ol>

<h1>4)   Implement explicit free list</h1>

The le explicit.c is your chance to turn your attention from utilization to throughput and work at making your allocator y.

The speci c features that your explicit free list allocator must support:

Block headers and in-place realloc (can copy from your implicit version)

Explicit free list managed as a doubly-linked list, uses rst 16 bytes of payload of freed block for next/prev pointers

Malloc searches explicit list of free blocks

Free immediately coalesces with any neighbor block(s) to right that are also free. Coalescing a pair of blocks must operate in O(1) time.

Note: the header should <strong>not</strong> be enlarged to add elds for the pointers. Since only free blocks are on the list, the more economical approach is to store those pointers in the otherwise unused payload. The minimum payload size will need be large enough to support this use.

The above bullets represent our requirements, all further decisions not speci cally mandated are yours to determine.

Answer the following question in your readme.txt:

<ol>

 <li>Compare the instruction counts of your implicit free list allocator versus your explicit free</li>

</ol>

list and summarize the gains in throughput.

If you are looking for some extra challenge, use callgrind to nd the hot spots and work to minimize them. Get help from gcc by editing the CFLAGS in your Make le and then add your own e orts to optimize what gcc cannot. How much improvement do your e orts make? Use your readme to tell us about your adventures in optimization — we’d love to hear about it!

<h1>5)   Final readme question</h1>

What would you like to tell us about your quarter in CS107? Of what are you particularly proud: the e ort you put into the quarter? the results you obtained? the process you followed? what you learned along the way? Tell us about it! We are not scoring this question, but wanted to give you a chance to share your closing thoughts before you move on to the next big thing.

<h1>Standard requirements for allocator</h1>

Requirements that apply to both allocators:

The interface should match the standard libc allocator. Carefully read the malloc man page for what constitutes a well-formed request and the required handling to which your allocator must conform. Note there are a few oddballs ( malloc(0) and the like) to take into account. Ignore esoteric details from the NOTES section of the man page.             There is no requirement on how to handle improper requests. If a client reallocs a stack address, frees an already freed pointer, overruns past the end of an allocated block, or other such incorrect usage, your response can be anything, including crashing or corrupting the heap. We will not test on these cases.

An allocated block must be at least as large as requested, but it is not required to be exactly that size. The maximum size request your design must accommodate is 1 &lt;&lt; 30 bytes (we originally set at INT_MAX, but that’s not even a multiple of 4 or 8 which makes it irritating if you need to round up). If asked for a block larger than the max size, your allocate should return NULL, just as with any request that cannot be serviced. Every allocated block must be aligned to an address that is a multiple of the ALIGNMENT constant (8). It’s your choice whether small requests are rounded up to some minimum size.

Your allocator cannot invoke any memory-management functions. By “memorymanagement” we speci cally mean those operations that allocate or deallocate memory, so no calls to malloc, realloc, free, calloc, sbrk, brk, mmap, or related variants. The use of other library functions is ne, e.g. memmove and memset can be used as they do not allocate or deallocate memory.

Your allocator may use a small amount of static global data, limited to at most 500 bytes. This restriction dictates the bulk of the heap housekeeping must be stored within the heap segment itself.

Your validate_heap function will be called from our test harness when checking correctness to verify the internal consistency of the heap data structures after servicing each request. This is here to help you with debugging. The validity check will not be made during any performance testing, so there is no concern about its e        ciency (or lack thereof).

<h1>Advice/FAQ</h1>

Don’t miss out on the companion advice page:  Go to advice/FAQ page (advice.html)


