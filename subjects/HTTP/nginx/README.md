Please carefully read the [main README.md](../../../README.md), which is stored in the benchmark's root folder, before following this subject-specific guideline.

# Notes about fuzzing performance with nginx
Please note that the performance of fuzzing the nginx service is not comparable to other fuzzer targets in ProFuzzBench such as LightFTP or forked-daapd.
Due to the size and matureness of nginx, a normal fuzzing process will not discover much of the source code and will almost certainly not find any crashes.
Because this target does not contain any PHP code, reverse proxied applications and so on, it will almost certainly hit the code paths for "400 Bad Request" or "404 Not Found" in the nginx code.
Anyway, this implementation can be used as a base for fuzzing other HTTP server software, either by using the reverse proxy of nginx or by installing another target in the `Dockerfile`.

Also, since HTTP is a stateless protocol, the usage of detailed test cases may not improve the fuzzing results, because the order of HTTP request should not impact the execution of server code. State Management like Sessions are not possible with the implemented fuzzers, as they would require the fuzzer to send a Session-Cookie with subsequent requests. Anyway, modifications to the fuzzers to support such behaviour are possible and can be implemented in this approach.

Additionally, it seems like the AFLNet parser for HTTP test cases is not yet able to support HTTP Requests with Body, since they would require a separator of `CRLF CRLF`, which is used by AFLNet to determine the end of a packet[^1].

# Fuzzing nginx server with AFLNet and AFLnwe
Please follow the steps below to run and collect experimental results for nginx.

## Step-1. Build a docker image
The following commands create a docker image tagged nginx. The image should have everything available for fuzzing and code coverage calculation.

```bash
cd $PFBENCH
cd subjects/HTTP/nginx
docker build . -t nginx
```

## Step-2. Run fuzzing
The following commands run 4 instances of AFLNet and 4 instances of AFLnwe to simultaenously fuzz the target in 60 minutes.

```bash
cd $PFBENCH
mkdir results-nginx

profuzzbench_exec_common.sh nginx 4 results-nginx aflnet out-nginx-aflnet "-P HTTP -D 200000 -m none -t 3000 -q 3 -s 3 -E -K" 3600 5 &
profuzzbench_exec_common.sh nginx 4 results-nginx aflnwe out-nginx-aflnwe "-D 200000 -m 1000 -t 3000 -K" 3600 5
```

## Step-3. Collect the results
The following commands collect the  code coverage results produced by AFLNet and AFLnwe and save them to results.csv.

```bash
cd $PFBENCH/results-nginx

profuzzbench_generate_csv.sh nginx 4 aflnet results.csv 0
profuzzbench_generate_csv.sh nginx 4 aflnwe results.csv 1
```

## Step-4. Analyze the results
The results collected in step 3 (i.e., results.csv) can be used for plotting. Use the following command to plot the coverage over time and save it to a file.

```
cd $PFBENCH/results-nginx

profuzzbench_plot.py -i results.csv -p nginx -r 4 -c 60 -s 1 -o cov_over_time.png
```

[^1]: <https://github.com/aflnet/aflnet/pull/71>