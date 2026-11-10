# Phase 3 — Comparison Writeup

Copy this file to `submission/phase3_comparison.md`, fill in every answer slot, and commit it:

```bash
cp phase-3-mapreduce-spark/comparison_template.md submission/phase3_comparison.md
```

**How to fill it in.** Each answer sits between a pair of `<!--answer:...-->` markers. Replace
the `TODO` line with your answer — **leave the markers themselves untouched**, they are how the
grader finds your answers. Anything you write outside the markers is ignored, so add extra
prose, tables or images freely.

Numbers in section 1 must **match `submission/phase3_report.json`** — copy them across, do not
retype from memory; they are checked against it.

---

## 1. Facts from your runs

Numbers only (e.g. `12`, `20.3`, `0.041`).

| | Value |
|---|---|
| Map tasks in the cloud job (`mapreduce.num_splits`) | <!--answer:num_map_tasks-->12<!--/answer--> |
| Reduce tasks in the cloud job (`mapreduce.num_reducers`) | <!--answer:num_reduce_tasks-->4<!--/answer--> |
| Cloud MapReduce wall time, seconds (`mapreduce.cloud_elapsed_s`) | <!--answer:cloud_elapsed_s-->48.6<!--/answer--> |
| Local single-process wall time, seconds (`mapreduce.local_elapsed_s`) | <!--answer:local_elapsed_s-->0.048<!--/answer--> |
| Spark wall time for the `collect`, seconds (`tfidf.spark_elapsed_s`) | <!--answer:spark_elapsed_s-->4.8<!--/answer--> |
| Number of **stages** in the Spark job (read off notebook 2b / 3a) | <!--answer:num_stages-->4<!--/answer--> |
| Task **retries** you counted in your `--chaos` run (0 if none happened) | <!--answer:chaos_retries-->12<!--/answer--> |

**Explain the timings.** The three wall times above cover the same 60 documents, but not the
same job: the single-process run and the cloud MapReduce are *the same word count*; Spark's
time is for *TF-IDF*, which is more work (two `reduceByKey`s and a `join`). Rank the three and
explain what each is spending its time on. Say which comparison is like-for-like, and what the
Spark number — a heavier job, on one machine — tells you about where the cloud job's time goes.
(~80 words)

<!--answer:timing_explanation-->
From fastest to slowest: local 0.048 s, Spark 4.8 s, cloud MapReduce 48.6 s. Local and cloud ran the same word count, so they are the pair to compare, and the cloud took about 1000 times longer. Counting the words takes 48 ms, so almost all of the 48.6 s went into Workflows steps, 16 authenticated HTTP calls, Cloud Run cold starts and around 65 Cloud Storage reads and writes. Spark did more work (TF-IDF) on a single machine and still finished ten times faster than the cloud job, which points to network round trips as the cost, not computation.
<!--/answer-->

**What the counts tell you.** From your stage count: how many shuffles did the Spark job have,
and which operations in `spark_tfidf.py` caused them? From your retry count: roughly what
fraction of the 16 task attempts was retried, and is that what `--chaos 0.3` should produce?
(~50 words)

<!--answer:counts_interpretation-->
Four stages means three shuffles: the reduceByKey in term_freq, the reduceByKey in doc_freq and the join in build_tfidf_rdd. In the chaos run I counted 12 retries for 16 tasks, so 28 attempts of which 12 failed (43%). With a failure probability of 0.3 per attempt I would expect about 7 retries (16 x 0.3 / 0.7). Twelve is on the high side, but possible with only 16 tasks.
<!--/answer-->

---

## 2. Spark: transformations, actions, lineage

**Transformations vs actions.** Go through `spark_tfidf.py` and the notebook's Spark cells.
Name each RDD operation and say whether it is a transformation or an action. What happened
when you ran `build_tfidf_rdd` — and what happened when you ran `collect`? (~60 words)

<!--answer:transformations_vs_actions-->
parallelize turns the document list into an RDD without computing anything. flatMap, reduceByKey (twice), map (three times) and join are transformations. collect is the only action, and toDebugString just prints the plan. Calling build_tfidf_rdd returned at once because it only recorded the lineage. Calling collect started one job with 4 stages and 12 tasks, which returned 5975 rows in 4.8 s.
<!--/answer-->

**Lineage and recovery.** Using the lineage you printed in 2b and the timeline in 3a: if the
worker holding one output partition of the *first* `reduceByKey` died, what would Spark have
to recompute, and what could it reuse? Name the stage or shuffle boundary where the recovery
starts. (~50 words)

<!--answer:lineage_recovery-->
The first reduceByKey produces ShuffledRDD[3]. If the worker holding one of its partitions died, Spark would rerun only that partition's task in the stage that starts at this first shuffle boundary. It would fetch the input again from the shuffle files that stage 0 wrote. Stage 0 (parallelize and flatMap) would only run again if those shuffle files were lost as well.
<!--/answer-->

---

## 3. Your cloud MapReduce, mapped onto Hadoop

Name the GCP service or file that played each role (one line each):

- **JobTracker** (schedules tasks, waits for the map phase, retries failures):
  <!--answer:role_jobtracker-->Cloud Workflows: the mr-wordcount workflow defined in workflow.yaml.<!--/answer-->
- **Workers** (run your map and reduce code):
  <!--answer:role_workers-->The Cloud Run function mr-worker (main.py), called once per map or reduce task.<!--/answer-->
- **HDFS** (holds inputs, shuffle files, outputs):
  <!--answer:role_hdfs-->The Cloud Storage bucket eurecomgpt-511211-eurecomgpt, with input/, shuffle/ and output/ folders under each job.<!--/answer-->
- **Partitioner** (decides which reducer owns a key):
  <!--answer:role_partitioner-->partition() in mapreduce.py, which computes zlib.crc32(word) % num_reducers and is called by run_map in main.py.<!--/answer-->

**The barrier.** There is no explicit "wait for all mappers" step in `workflow.yaml`. Where,
exactly, is the barrier between the map and reduce phases — which construct provides it?
(~30 words)

<!--answer:barrier_location-->
The barrier is the map_phase parallel step. A parallel step only ends when all of its branches have returned, retries included, and reduce_phase is the step after it, so no reducer can start before the last mapper is done.
<!--/answer-->

**Idempotence.** The workflow retries a task that failed. Why is that safe with the worker as
written, and what would go wrong if `run_map` *appended* its pairs to the shuffle file instead
of overwriting it? (~60 words)

<!--answer:idempotence-->
Every task writes to fixed paths (shuffle/split-NNN/part-R.json and output/part-R.json) and overwrites what is there. A retried task reads the same input, computes the same pairs and writes the same files, so running it twice gives the same result as running it once. If run_map appended instead, a task that failed after writing (a timeout, for example) would write its pairs a second time on retry, and the reducers would count those words twice.
<!--/answer-->

**The chaos run.** What did you see in the execution log and in Logs Explorer when you ran
with `--chaos 0.3`? Did the job still succeed, and was the final answer still identical to the
local reference? (~50 words)

<!--answer:chaos_observations-->
In the execution view, some map and reduce tasks failed with a 503 and the message "simulated worker crash", then succeeded on a later attempt after the backoff. The worker does not log that message, so in Logs Explorer I found the crashes as request entries with status 503, 12 of them. The job still succeeded in 48.6 s (48.7 s without chaos) and found 2221 words, identical to the local reference.
<!--/answer-->
