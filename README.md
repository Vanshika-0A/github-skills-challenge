# GitHub Challenge

<img src="https://octodex.github.com/images/Professortocat_v2.png" align="right" height="200px" />

Hey there!

Your challenge is ready.
Follow the instructions provided for this challenge and complete the required tasks in this repository.

Make sure your work is committed and pushed to your repository before submission.

Good luck!


---

&copy; 2025 GitHub &bull; [Code of Conduct](https://www.contributor-covenant.org/version/2/1/code_of_conduct/code_of_conduct.md) &bull; [MIT License](https://gh.io/mit)

## AIOps Scenario

For this assessment I worked with a small AIOps simulation for a payment-service.
The service produces some telemetry data (response time, CPU usage, memory usage)
along with log messages. The idea is that instead of someone manually checking logs
all day to see if something's wrong, this pipeline automatically looks at the data,
figures out if something unusual is happening, and turns that into an "event" that
gets passed through a producer/topic/consumer setup (like a mini version of something
like Kafka, but simulated in plain Python).

Basically the problem this solves is: how do you catch things like slow response
times or errors automatically, without a human staring at logs 24/7.

## Operational Data

The data is in `data/service_data.json`. It's 10 records, one per minute, for a
service called payment-service.

Fields I found:
- response_time_ms, cpu_percent, memory_percent → these are the metrics (numbers)
- log_level and message → these are the log info
- timestamp → ISO format, one minute apart each time

Looking through the data manually:
- Most records (8 out of 10) look totally normal — response time around 120-150ms,
  CPU 42-50%, memory 51-57%, log_level is INFO.
- Two records stand out — at 10:05 and 10:06 the response time jumps to 610ms and
  640ms, CPU goes up to 75% and 94%, memory goes up to 70% and 91%, and the log_level
  switches to ERROR with messages about timeouts. Right after that (10:07) it goes
  back to normal, so it looks like a short spike/incident rather than something
  ongoing.

## Anomaly Detection

The detector is in src/anomaly_detector.py. It checks each record against some
thresholds — response time over 500ms, CPU over 80%, memory over 80%, or if the
log level is an error.

When I ran it on the data, it correctly picked up both weird records (10:05 and
10:06) and gave reasons for why each one was flagged. It didn't miss anything and
it didn't wrongly flag any of the normal records, so it matched what I found just
by eyeballing the data.

One thing I noticed — 10:05 didn't get flagged for CPU/memory because those were
75% and 70%, still under the 80% cutoff, so it only got flagged for response time
and the error log. 10:06 was worse and hit all the thresholds.

Limitation: the thresholds are just fixed numbers. That works fine for this small
dataset but wouldn't really work well in real life since every service has a
different "normal" — a fixed 80% CPU cutoff might be way too sensitive for one
service and not sensitive enough for another. A better approach might be to look
at how much a value deviates from that service's own recent average instead of
using one fixed number for everyone.

## How the Event Flow Works

- event_producer.py — takes an anomaly and publishes it
- event_topic.py — just an in-memory list acting like a topic/queue
- event_consumer.py — reads whatever's sitting in the topic
- aiops_pipeline.py — this is the main script that ties everything together:
  loads the data, runs it through the detector, publishes anomalies, then
  reads them back out at the end and prints the final result

## Running It / Final Result

Running:
gives this output:Records processed: 10
Anomalies detected: 2
Events consumed: 2

Detected Events:

Service: payment-service
Timestamp: 2026-09-20T10:05:00
Type: ANOMALY
Reasons: High response time, Error log detected

Service: payment-service
Timestamp: 2026-09-20T10:06:00
Type: ANOMALY
Reasons: High response time, High CPU utilization, High memory utilization, Error log detected

So both anomalies made it all the way through the pipeline — detected, published,
and successfully picked up by the consumer at the end.

## Bugs I Found and Fixed

**Bug 1 — error logs weren't being picked up**
In anomaly_detector.py, the code was checking if log_level == "WARNING", but if
you look at the actual data, log_level is only ever "INFO" or "ERROR" — there's
no "WARNING" anywhere. So that check could literally never be true. I changed it
to check for "ERROR" instead, since that's what the data actually has. After the
fix, both anomalies started showing up with "Error log detected" in their reasons.

**Bug 2 — consumer wasn't getting any events**
In aiops_pipeline.py, the producer and consumer were each given their own separate
EventTopic object (one called "service-events", the other "anomaly-events"). Since
they were two completely different objects, anything the producer published never
actually reached the consumer — it was publishing into one topic and the consumer
was reading from a totally different, empty one. I fixed this by creating just one
shared EventTopic and passing that same one to both the producer and the consumer.
After the fix, "Events consumed" went from 0 to 2, matching the number of detected
anomalies.

## Tests

Ran the test suite with:python -m pytest tests/
 tests/calculations_test.py .... [ 50%]
tests/test_aiops_pipeline.py .... [100%]
8 passed in 0.07s

## How to Reproduce This

1. Fork the repo and open it in a GitHub Codespace (default settings are fine)
2. Once it loads, install requirements if needed: pip install -r requirements.txt
3. Run the pipeline: python src/aiops_pipeline.py
4. Run the tests: python -m pytest tests/
5. You should see 2 anomalies detected, 2 events consumed, and all 8 tests passing.