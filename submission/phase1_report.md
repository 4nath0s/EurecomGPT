# Phase 1 — Comparison Report

Copy this file to `submission/phase1_report.md`, fill in every answer slot, and commit it:

```bash
cp phase-1-echo-bot/report_template.md submission/phase1_report.md
```

**How to fill it in.** Each answer sits between a pair of `<!--answer:...-->` markers. Replace
the `TODO` line with your answer — **leave the markers themselves untouched**, they are how the
grader finds your answers. Anything you write outside the markers is ignored, so add extra
prose, tables or images freely.

Numbers must **match `submission/phase1_report.json`** exactly — that file is produced by
`measure.py` and is what your figures are checked against. Copy them across; do not retype from
memory.

---

## 1. Setup facts

**Your image size**, in MB, from `gcloud artifacts docker images list` (number only, e.g. `142`):

<!--answer:image_size_mb-->
47.7
<!--/answer-->

**The custom VPC and subnet you created** (names only):

<!--answer:vpc_name-->
echo-net
<!--/answer-->

<!--answer:subnet_name-->
echo-subnet
<!--/answer-->

---

## 2. Measurements

Copy these from `submission/phase1_report.json`. Cold is `cold_ms`; warm median is
`stats.median`. Numbers only, in milliseconds (e.g. `812.4`).

| Platform | Cold (ms) | Warm median (ms) |
|---|---|---|
| IaaS VM | <!--answer:vm_cold_ms-->257.5<!--/answer--> | <!--answer:vm_warm_median_ms-->211.0<!--/answer--> |
| Cloud Run | <!--answer:cloudrun_cold_ms-->2231.2<!--/answer--> | <!--answer:cloudrun_warm_median_ms-->128.7<!--/answer--> |
| Cloud Function | <!--answer:function_cold_ms-->1114.2<!--/answer--> | <!--answer:function_warm_median_ms-->127.7<!--/answer--> |

Include your histogram or plot of the warm distribution below (an image, or the ASCII chart
`measure.py` prints). This is read by a human, not the grader:

<!--answer:latency_plot-->
```
Phase 1 — deployment measurements
------------------------------------------------------------
[OK  ] vm        cold=  257.5ms  warm_median=  211.0ms
            ########################################
[OK  ] cloudrun  cold= 2231.2ms  warm_median=  128.7ms
            ########################
[OK  ] function  cold= 1114.2ms  warm_median=  127.7ms
            ########################
------------------------------------------------------------
```
The VM has no real cold start: its container never stops, so only the first connection adds a few milliseconds.
<!--/answer-->

---

## 3. Per-platform analysis

**Scaling behaviour.** Imagine requests arriving one at a time, then ten at once, then a
hundred. For each of the three platforms, answer concretely:

- Does anything get created automatically to absorb the load — and if so, *what* exactly?
  (One platform here creates nothing at all.)
- What is the hard limit, and where does it come from — a flag you passed yourself, or the
  size of the machine you picked?

(~80 words. Your `--machine-type=e2-micro` and `--max-instances=3` are both concrete answers
to the second question.)

<!--answer:scaling_behaviour-->
VM: nothing is created. One e2-micro (2 shared vCPUs, 1 GB RAM) runs one container, so ten or a hundred simultaneous requests queue on that single process. The limit is the machine size I chose.

Cloud Run: new instances of my image start automatically, each taking up to 80 concurrent requests (default). The limit is --max-instances=3, my own flag; beyond it, requests queue, then get a 429.

Function: same mechanism with the buildpack image, but gen-2 defaults: one request per instance, up to 100 instances, since I set no cap.
<!--/answer-->

**Deployment effort.** What did you have to build, configure and maintain to get each of the
three running? (~100 words)

<!--answer:deployment_effort-->
VM (most work): Dockerfile, image build and push, custom VPC, subnet, two tagged firewall rules, a tagged VM, an IAM grant so it can pull, then SSH, Docker install, credential helper and docker run with a restart policy. I now own OS and Docker patching, uptime and the billed external IP, and it serves plain HTTP only.

Cloud Run: the same image plus one gcloud run deploy; HTTPS, scaling and host maintenance are Google's.

Function (least): source and an entry point, one command. Buildpacks build the container, so I maintain only code and requirements, but lose control of the base image and server.
<!--/answer-->

---

## 4. Why do two of your three numbers look alike?

With Cloud Run and your Function both deployed, run `gcloud run services list` and paste the
**complete output** here, unedited:

<!--answer:run_services_list-->
```
✓
SERVICE: echo
REGION: us-central1
URL: https://echo-689838645589.us-central1.run.app
LAST DEPLOYED BY: service-689838645589@gcf-admin-robot.iam.gserviceaccount.com
LAST DEPLOYED AT: 2026-10-26T22:25:55.465142Z

✓
SERVICE: echo-bot
REGION: us-central1
URL: https://echo-bot-689838645589.us-central1.run.app
LAST DEPLOYED BY: antoine.danaels@eurecom.fr
LAST DEPLOYED AT: 2026-10-26T22:17:35.233981Z
```
<!--/answer-->

Now explain what that output tells you about how your Function is actually being run, why its
cold start resembles Cloud Run's so closely, and what — if anything — genuinely differs between
the two deployment paths. (~120 words)

<!--answer:convergence_explanation-->
My Function echo appears as a Cloud Run service next to echo-bot, with a run.app URL, deployed by gcf-admin-robot (the Cloud Functions service agent), not by me. gcloud functions deploy --gen2 is a front end to Cloud Run: Cloud Build packaged my source into a container (Python buildpack + Functions Framework), and the service agent deployed it as a Cloud Run service.

Both cold starts therefore follow the same path: with zero instances, Cloud Run schedules one, starts the container and waits for it to listen. Hence 1–2 s cold against ~128 ms warm for both, and near-identical warm medians (128.7 vs 127.7 ms). With one cold sample each, the 2× gap (2231.2 vs 1114.2 ms) cannot be attributed to the deployment path; resources do not explain it either, since the faster function has a sixth of the CPU.

What differs is upstream: who builds the image (my Dockerfile vs buildpacks on a Google-maintained python311 base image), what serves requests (Flask vs Functions Framework), the defaults (concurrency 80 vs 1, 1 vCPU / 512 MiB vs 0.17 vCPU / 256 MB, max 3 vs 100 instances), and who manages the service.
<!--/answer-->
