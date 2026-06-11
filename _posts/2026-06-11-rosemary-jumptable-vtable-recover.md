## Rosemary jumpable and vtable recovery

After two months of development, Rosemary can finally restore jumptable and vtable.

On ARM64, restoring jumptable and vtable involves identifying load/stop pages via IR.

Running Status:



```shell
data_flow_round: 1
data_flow_round: 2
 ----- elf memory start ------
[elf memory] elf's binary buffer: 24582248, 23 MB
[elf memory] elf's entry points: 96962, size: 32 memory: 2 MB
[elf memory] elf's entry hashmap: 96962 size: 16, memory: 1 MB
[elf memory] elf's jd_pc_addr: 86131, size: 16, memory: 1 MB
[elf memory] elf's jd_addr: 122018, size: 16, memory: 1 MB

[elf region] pc range: 0 -> 16eb220
[elf memory] elf's region bitmap: 2 MB
[elf memory] elf's basic block count: 825228, size: 168, memory: 132 MB, includes defs/uses/livein/liveout
[elf memory]       basic block's defs count: 825228, size: 16, memory: 12 MB
[elf memory]       basic block's uses count: 825228, size: 16, memory: 12 MB
[elf memory]       basic block's livein count: 825228, size: 16, memory: 12 MB
[elf memory]       basic block's liveout count: 825228, size: 16, memory: 12 MB
[elf memory] elf's edge count: 2537258, size: 8, memory: 19 MB
[elf memory] elf's expression count: 15195444, size: 32, memory: 463 MB
[elf memory] elf's jd_elf_func_ins count: 4544973, size: 16, memory: 69 MB
[elf memory] elf's def_exp_ids_arr: 3221610, size: 4, memory: 12 MB
[elf memory] elf's use_exp_ids_arr: 2546554, size: 4, memory: 9 MB
[elf memory] elf's phi_args_arr: 6445660, size: 4, memory: 24 MB
[elf memory] elf's invoke_args_arr: 5175409, size: 4, memory: 19 MB
[elf memory] elf's pc_map_ins: 12015888, size: 4, memory: 45 MB
[elf memory] elf's functions: 52458, size: 72, memory: 3 MB
[elf memory] elf's jumptable: 521, size: 72, memory: 0 MB
[elf memory] elf's jumptable target: 51913, size: 4, memory: 0 MB
[elf memory] elf's vtable: 2435, size: 40, memory: 0 MB
[elf memory] elf's vtable target: 15499, size: 4, memory: 0 MB
[elf memory] elf's phi_arr: 2540117, size: 20, memory: 48 MB
[elf memory] elf's collapsed phi count: 652440
[elf memory] elf's xref: 1638108, size: 12, memory: 18 MB
[elf memory] elf's stack slot: 391747, size: 20, memory: 7 MB
[elf memory] elf's instruction: 4729638, size: 24, memory: 108 MB
[elf memory] elf's region: 3, size: 1952, memory: 0 MB
[elf memory] elf's template count: 6000, size: 68, memory: 0 MB
 ----- elf memory end ------

 ----- elf info start ------
[elf info] elf's execute region 0: 0 - 16eb220
[elf info] elf's execute region 1: 16ef220 - 17550b0
[elf info] elf's execute region 2: 175c000 - 17d9618
[elf info] elf's entry address size: 96962
 ----- elf info end ------
	Command being timed: "./build/rosemary"
	User time (seconds): 2.42
	System time (seconds): 0.13
	Percent of CPU this job got: 99%
	Elapsed (wall clock) time (h:mm:ss or m:ss): 0:02.57
	Average shared text size (kbytes): 0
	Average unshared data size (kbytes): 0
	Average stack size (kbytes): 0
	Average total size (kbytes): 0
	Maximum resident set size (kbytes): 1092832
	Average resident set size (kbytes): 0
	Major (requiring I/O) page faults: 6
	Minor (reclaiming a frame) page faults: 76776
	Voluntary context switches: 1
	Involuntary context switches: 252
	Swaps: 0
	File system inputs: 0
	File system outputs: 0
	Socket messages sent: 0
	Socket messages received: 0
	Signals delivered: 0
	Page size (bytes): 16384
	Exit status: 0
```



running on Macmini M4, A 23M ELF can complete the analysis and modeling of the recovery jumptable and vtable in about 2 seconds.