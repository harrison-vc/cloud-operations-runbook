# Triage Flowchart

[Start] -> (External Check: Status 200?)
           |-- No --> [Network/WAF Issue]
           |-- Yes --> (Local Check: Success?)
                       |-- No --> [Application/Process Issue]
                       |-- Yes --> (Resource Check: Free space/CPU?)
                                   |-- No --> [Compute/Disk Issue]
                                   |-- Yes --> [Upstream/Database Issue]
