./cockroach workload init kv --zipfian
./cockroach workload run kv --concurrency 3 --duration 300s --zipfian 

./cockroach workload init ycsb --workload F --drop --insert-count 1000000
./cockroach workload run ycsb --workload F --concurrency 3 --duration 300s


./workload run querybench --query-file b1_query.sql --db mybench_b1 --duration 120s --concurrency 3
./workload run querybench --query-file b1_query.sql --db mybench_b1 --duration 120s --concurrency 6
./workload run querybench --query-file b10_query.sql --db mybench_b10 --duration 120s --concurrency 1
./workload run querybench --query-file b10_query.sql --db mybench_b10 --duration 120s --concurrency 3
