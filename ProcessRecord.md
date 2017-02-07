# ProcessRecord


	Full information about a particular process that is currently running.

	
保存进程的信息。


修改进程的 adj


```
int modifyRawOomAdj(int adj) {
    if (hasAboveClient) {
        // If this process has bound to any services with BIND_ABOVE_CLIENT,
        // then we need to drop its adjustment to be lower than the service's
        // in order to honor the request.  We want to drop it by one adjustment
        // level...  but there is special meaning applied to various levels so
        // we will skip some of them.
        if (adj < ProcessList.FOREGROUND_APP_ADJ) {
            // System process will not get dropped, ever
        } else if (adj < ProcessList.VISIBLE_APP_ADJ) {
            adj = ProcessList.VISIBLE_APP_ADJ;
        } else if (adj < ProcessList.PERCEPTIBLE_APP_ADJ) {
            adj = ProcessList.PERCEPTIBLE_APP_ADJ;
        } else if (adj < ProcessList.CACHED_APP_MIN_ADJ) {
            adj = ProcessList.CACHED_APP_MIN_ADJ;
        } else if (adj < ProcessList.CACHED_APP_MAX_ADJ) {
            adj++;
        }
    }
    return adj;
}
```