## Redshift query optimization

1. What is the query? 
2. Investigate the query plan by using `EXPLAIN`
3. Where the query is executed from? SQL client or application?
4. What is the server version? Is it the latest release? Refer to https://docs.aws.amazon.com/redshift/latest/mgmt/cluster-versions.html
5. There are two query types of cache. Ensure which one that we hit:
    1. `Compilation cache` where it uses a serverless compilation farm outside of the cluster. The compiled code is also stored off-cluster
    2. `Query result cache` where for the same query the result is saved into a cache for fast retrieval. can be disabled by using `SET enable_result_cache_for_session TO OFF;`
6. Query execution:
    `runtime = plan_time + wait_time + exec_time`
    
    Use the following query:
    ```
    select stlw.userid, stlw.query, stlw.total_queue_time as total_queue_time_us, stlw.total_exec_time as total_exec_time_us, svlc.total_compile_time_us 
    from (select query,min(starttime) compile_begin,max(endtime) compile_end,datediff(us,min(starttime),max(endtime)) as total_compile_time_us, sum(datediff(us,starttime,endtime)) as actual_compile_time_us from svl_compile group by 1) svlc 
    join stl_wlm_query stlw on svlc.query=stlw.query 
    join (select query,xid,aborted from stl_query) stlq on stlw.query=stlq.query where stlw.query= <query_id>;
    ```
    


7. Check `SVL_COMPILE` table:

    ```
    select userid, xid,  pid, query, segment, locus,  
    datediff(ms, starttime, endtime) as duration, compile 
    from svl_compile 
    where query in (<query_id>)
    order by query, segment;
    ```
    If `COMPILE = 1` then there is a compilation. Check the duration! 
    
8. Check `SVL_QUERY_QUEUE_INFO` table:
    ```
    select * from SVL_QUERY_QUEUE_INFO
    where query in (<query_id>);
    ```
    How long does it take in queue? If no entry then it does not get queued.
    
9. Check the queue average waiting time in the WLM, is it a loaded queue?
    ```
    select a.service_class as svc_class, b.name , count(*) count_query,
    avg(datediff(microseconds, queue_start_time, queue_end_time)) as avg_queue_time,
    avg(datediff(microseconds, exec_start_time, exec_end_time )) as avg_exec_time
    from stl_wlm_query a
    join STV_WLM_SERVICE_CLASS_CONFIG b on a.service_class=b.service_class
    where a.service_class > 4
    group by a.service_class, b.name
    order by a.service_class;
    ```
    
    

        

