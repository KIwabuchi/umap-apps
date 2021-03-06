
# BFS

## Generate Edge List Using an R-MAT Generator

```
./rmat_edge_generator/generate_edge_list

-o [out edge list file name (required)]

-s [seed for random number generator; default is 123]

-v [SCALE; The logarithm base two of the number of vertices; default is 17]

-e [#of edges; default is 2^{17} x 16]

-a [initiator parameter A; default is 0.57]

-b [initiator parameter B; default is 0.19]

-c [initiator parameter C; default is 0.19]

-r [if true, scrambles edge IDs; default is true]

-u [if true, generates edges for both directions; default is true]
```

* As for the initiator parameters, see [Graph500, 3.2 Detailed Text Description](https://graph500.org/?page_id=12#sec-3_2) for more details.

### Generate Graph 500 Inputs

```bash
./rmat_edge_generator/generate_edge_list -o /mnt/ssd/edge_list -v 20 -e $((2**20*16))
````

* This command generates a edge list file (/mnt/ssd/edge_list) which contains the edges of a SCALE 20 graph.
* In Graph 500, the number of edges of a graph is \#vertices x 16 (16 is called 'edge factor') as an undirected graph.
* Note that \#edges generated by this generator is \#vertices x 16 x 2 if -u option (generates edges for both directions) is true, which is default.
* This is a multi-threads (OpenMP) program. You can control the number of threads using the environment variable OMP\_NUM\_THREADS.


## Ingest Edge List (construct CSR graph)

```bash
./ingest_edge_list -g /mnt/ssd/csr_graph_file /mnt/ssd/edge_list1 /mnt/ssd/edge_list2
```

* Load edge data from files /mnt/ssd/edge\_list1 and /mnt/ssd/edge\_list2 (you can specify an arbitrary number of files). A CSR graph is constructed in /mnt/ssd/csr\_graph\_file.
* Each line of input files must be a pair of source and destination vertex IDs (unsigned 64bit number).
* This program treats inputs as a directed graph, that is, it does not ingest edges for both directions.
* This is a multi-threads (OpenMP) program. You can control the number of threads using the environment variable OMP\_NUM\_THREADS.
* As for real-world datasets, [SNAP Datasets](http://snap.stanford.edu/data/index.html) is popular in the graph processing community. Please note that some datasets in SNAP are a little different. For example, the first line is a comment; you have to delete the line before running this program.


## Run BFS

```bash
./run_bfs -n [#of vertices] -m [#of edges] -g [/path/to/graph_file] -s
```

* You can get #of vertices and #of edges by running ingest\_edge\_list.
* If '-s' is specified, the program uses system mmap instead of umap.
* The interface to the umap runtime library configuration is controlled by environment variables, see [Umap Runtime Environment Variables](https://llnl-umap.readthedocs.io/en/develop/environment_variables.html).
* This is a multi-threads (OpenMP) program. You can control the number of threads using the environment variable OMP\_NUM\_THREADS. It uses the static schedule.


## Tips for Running Benchmark (on large-scale)
* The size of generated edge lists could be larger than the constructed CSR graph by a few times. As the rmat\_edge\_generator writes edges to files sequentially, you should be able to directly generate edge lists to a parallel file systems without an unreasonable execution time.
* On the other hand, ingest\_edge\_list constructs a CSR graph causing a lot of random writes to a file mapped region (the location of the file is specified by -g option). We highly recommend that you should construct a graph on DRAM space, e.g., tmpfs, although you can still read input edge list files from a parallel file system.


### Example
Run BFS on a SCALE 20 R-MAT graph using 4 threads and system mmap.
```bash
env OMP_NUM_THREADS=4 ./rmat_edge_generator/generate_edge_list -o /mnt/ssd/edge_list -v 20 -e $((2**20*16))
env OMP_NUM_THREADS=4 ./ingest_edge_list -g /dev/shm/csr_graph_file /mnt/ssd/edge_list*
mv /dev/shm/csr_graph_file /mnt/ssd/csr_graph_file
sudo sync
sudo echo 3 > /proc/sys/vm/drop_caches # drop page cache
env OMP_NUM_THREADS=4 ./run_bfs -n $((2**20)) -m $((2**20*16*2)) -g /mnt/ssd/csr_graph_file -s
```
