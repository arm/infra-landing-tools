# Code Region Statistics and Java Code Region Mapping Tools

spe-region.py and region_map.py are two separate tools. 

---

## spe-region.py

Performs statistics on region info from perf SPE data:
- Calculate code heat for each region
- Calculate branch jump relations between regions

The outputs are serial of PNG files. 
Touch ratio means how many small code chunks in a 2 MB region are touched. 

If you find active region count is much larger than region buffer count
of your Arm CPU model, and the touch ratio are really low, PGO/AutoFDO
may help in performance improvement. 

### How to run

1. Capture SPE data (tune sleep duration based on desired data size):
   ```bash
   perf record --no-switch-events -e 'arm_spe_0/jitter=1/' -c 2048 -N -p $pid -- sleep 2
   ```
2. Prepare the SPE parser tool:
   - Install `tools/spe_parser` from https://gitlab.arm.com/telemetry-solution/telemetry-solution
3. Place `perf.data` (from step 1) in this folder.
4. Run the tool:
   ```bash
   ./spe-region.py perf.data
   ```

---

## region_map.py

Maps Java compiled code to 2MB region spaces.

### How to run

1. Preferred (if your JVM supports it):  
   Generate the perf map:
   ```bash
   jcmd <pid> Compiler.perfmap
   ```
   If not supported, use steps a–d below.

2. Alternative (using perf-map-agent):
   - a. Get perf-map-agent: https://github.com/jvm-profiling-tools/perf-map-agent.git
   - b. Apply the following change to `src/c/perf-map-agent.c`:
     ```diff
     diff --git a/src/c/perf-map-agent.c b/src/c/perf-map-agent.c
     index a5dea76..561b025 100644
     --- a/src/c/perf-map-agent.c
     +++ b/src/c/perf-map-agent.c
     @@ -368,3 +368,35 @@ Agent_OnAttach(JavaVM *vm, char *options, void *reserved) {
           return 0;
       }
       
     +JNIEXPORT jint JNICALL Agent_OnLoad(JavaVM *vm, char *options, void *reserved) {
     +    open_map_file();
     +
     +    if (options == NULL) {
     +        options = "";
     +    }
     +    unfold_simple = strstr(options, "unfoldsimple") != NULL;
     +    unfold_all = strstr(options, "unfoldall") != NULL;
     +    unfold_inlined_methods = strstr(options, "unfold") != NULL || unfold_simple || unfold_all;
     +    print_method_signatures = strstr(options, "msig") != NULL;
     +    print_source_loc = strstr(options, "sourcepos") != NULL;
     +    dotted_class_names = strstr(options, "dottedclass") != NULL;
     +    clean_class_names = strstr(options, "cleanclass") != NULL;
     +    annotate_java_frames = strstr(options, "annotate_java_frames") != NULL;
     +
     +    bool use_semicolon_unfold_delimiter = strstr(options, "use_semicolon_unfold_delimiter") != NULL;
     +    unfold_delimiter = use_semicolon_unfold_delimiter ? ";" : "->";
     +
     +    debug_dump_unfold_entries = strstr(options, "debug_dump_unfold_entries") != NULL;
     +
     +    jvmtiEnv *jvmti;
     +    (*vm)->GetEnv(vm, (void **)&jvmti, JVMTI_VERSION_1);
     +    enable_capabilities(jvmti);
     +    set_callbacks(jvmti);
     +    set_notification_mode(jvmti, JVMTI_ENABLE);
     +    (*jvmti)->GenerateEvents(jvmti, JVMTI_EVENT_DYNAMIC_CODE_GENERATED);
     +    (*jvmti)->GenerateEvents(jvmti, JVMTI_EVENT_COMPILED_METHOD_LOAD);
     +    set_notification_mode(jvmti, JVMTI_DISABLE);
     +    close_map_file();
     +
     +    return 0;
     +}
     ```
   - c. Build perf-map-agent:
     ```bash
     cmake .
     make
     ```
   - d. Run your Java application and generate the perf map:
     ```bash
     java -agentpath:/path/to/libperfmap.so -jar your-application.jar
     ./bin/create-java-perf-map.sh <pid>
     # Output: /tmp/perf-<pid>.map
     ```

3. Generate the SVG mapping with `region_map.py`:
   ```bash
   ./region_map.py /tmp/perf-<pid>.map perf-<pid>.svg
   ```
