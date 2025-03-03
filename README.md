## ZENIGMA TASK
#### Repository for zenigma tasks


### THIS IS THE CONTENT OF SH SCRIPT FOR HELLO ZENIGMA
    vi hello_zenigma.sh
    ### !/bin/bash
    
    ### Write "Hello Zenigma" to zenigma.log for very 15 seconds
    while true
    do
    echo "Hello Zenigma" >> zenigma.log
    sleep 15
    done


### THIS IS THE CONTENT OF HELLO_ZENIGMA.SERVİCE FOR MAKING IT AS A DAEMON PROCESS

#### First we need to identify new service under /etc/systemd/system/hello_zenigma.service
    vi /etc/systemd/system/hello_zenigma.service
    [Unit]
    Description=Hello Zenigma Daemon
    
    [Service]
    ExecStart=/bin/bash /root/hello_zenigma.sh
    StandardOutput=append:/root/zenigma.log
    StandardError=append:/root/zenigma.log
    Restart=always
    WorkingDirectory=/root
    
    [Install]
### THIS IS THE CONTENT OF SH SCRIPT FOR CHECKING HELLO_ZENIGMA.SH COUNT
    ### !/bin/bash
    
    ### Configuration
    LOG_FILE="/root/zenigma.log"
    SERVICE_NAME="hello_zenigma.service"
    MAX_RECORDS=20
    KILLER_LOG="/root/killer.log"
    
    ### Function to log messages to killer.log
    log_to_killer() {
    echo "[$(date +"%Y-%m-%d %H:%M:%S")] $1" >> "$KILLER_LOG"
    echo "[$(date +"%Y-%m-%d %H:%M:%S")] $1"  ### Also display in terminal
    }
    
    ### Function to check if service is active
    is_service_active() {
    if systemctl is-active --quiet "$SERVICE_NAME"; then
    return 0  ### True, service is active
    else
    return 1  ### False, service is inactive
    fi
    }
    
    ### Function to count "Hello Zenigma" entries in log file
    count_hello_zenigma() {
    if [ -f "$LOG_FILE" ]; then
    ### Count lines containing "Hello Zenigma" in the log file
    count=$(grep -c "Hello Zenigma" "$LOG_FILE" 2>/dev/null || echo 0)
    echo "$count"
    else
    echo "0"
    fi
    }
    
    ### Function to restart service and reset log
    reset_service() {
    local current_count=$(count_hello_zenigma)
    log_to_killer "THRESHOLD REACHED - Line count: $current_count/$MAX_RECORDS - Initiating reset procedure"
    
        ### Stop the service
        log_to_killer "Stopping $SERVICE_NAME"
        systemctl stop "$SERVICE_NAME"
        
        ### Wait for the service to fully stop (max 10 seconds)
        local wait_count=0
        while is_service_active && [ $wait_count -lt 10 ]; do
            log_to_killer "STATUS: Waiting for service to stop completely..."
            sleep 1
            wait_count=$((wait_count + 1))
        done
        
        if is_service_active; then
            log_to_killer "WARNING: Service did not stop gracefully, forcing termination"
            ### Find and kill the process more aggressively if needed
            pkill -f "hello_zenigma.sh"
            sleep 1
        fi
        
        ### Delete the log file and report status
        if [ -f "$LOG_FILE" ]; then
            log_to_killer "ACTION: Deleting $LOG_FILE"
            rm -f "$LOG_FILE"
            if [ ! -f "$LOG_FILE" ]; then
                log_to_killer "STATUS: File deletion successful"
            else
                log_to_killer "ERROR: File deletion failed"
            fi
        else
            log_to_killer "STATUS: No log file found to delete"
        fi
        
        ### Start the service again
        log_to_killer "Starting $SERVICE_NAME"
        systemctl start "$SERVICE_NAME"
        
        ### Wait for service to start (max 5 seconds)
        wait_count=0
        while ! is_service_active && [ $wait_count -lt 5 ]; do
            log_to_killer "STATUS: Waiting for service to start..."
            sleep 1
            wait_count=$((wait_count + 1))
        done
        
        if is_service_active; then
            log_to_killer "STATUS: Service successfully restarted - Reset complete"
        else
            log_to_killer "ERROR: Failed to restart service - Reset incomplete"
        fi
    }
    
    ### Check if script is run as root
    if [ "$EUID" -ne 0 ]; then
    echo "This script must be run as root to control systemd services"
    exit 1
    fi
    
    ### Main monitoring loop
    log_to_killer "========== MONITOR STARTED =========="
    log_to_killer "CONFIGURATION: Target count threshold: $MAX_RECORDS"
    log_to_killer "CONFIGURATION: Monitoring file: $LOG_FILE"
    log_to_killer "CONFIGURATION: Monitoring service: $SERVICE_NAME"
    
    ### Initialize last count
    last_count=0
    
    while true; do
    ### Check if service is active
    if ! is_service_active; then
    log_to_killer "STATUS: Service $SERVICE_NAME is inactive"
    log_to_killer "ACTION: Starting service $SERVICE_NAME"
    systemctl start "$SERVICE_NAME"
    sleep 2
    
            if ! is_service_active; then
                log_to_killer "ERROR: Failed to start service - Will retry in 30 seconds"
                sleep 30
                continue
            else
                log_to_killer "STATUS: Service successfully started"
            fi
        fi
        
        ### Count "Hello Zenigma" entries
        current_count=$(count_hello_zenigma)
        
        ### Log count every minute or if it changed
        current_minute=$(date +"%M")
        if [ "$current_count" != "$last_count" ] || [ "$current_minute" != "$last_logged_minute" ]; then
            log_to_killer "COUNT: Hello Zenigma entries: $current_count/$MAX_RECORDS"
            last_count=$current_count
            last_logged_minute=$current_minute
        fi
        
        ### Check if count exceeds threshold
        if [ "$current_count" -ge "$MAX_RECORDS" ]; then
            reset_service
            last_count=0
        fi
        
        ### Wait before checking again
        sleep 5
    done
### DEFINING ZENIGMA_MONITOR.SH AS A CRON JOB

    sudo crontab -e
    
    */5 * * * * /root/zenigma_monitor.sh >> /root/cron_monitor.log 2>&1
    
    crontab -l
#### Addıng " >>/root/cron_monitor.log 2>&1" for seeing if any error occured while job is executing every 5 minutes

