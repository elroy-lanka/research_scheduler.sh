[research_scheduler.sh.sh](https://github.com/user-attachments/files/31437002/research_scheduler.sh.sh)
# research_scheduler.sh#!/bin/bash

# ==========================================================
# University Research Cluster Job Scheduler
# Operating Systems Assessment - Task 2
# ==========================================================

# Data files
JOB_QUEUE="job_queue.txt"
COMPLETED_JOBS="completed_jobs.txt"
SCHEDULER_LOG="scheduler_log.txt"

# Round Robin time quantum
TIME_QUANTUM=5

# Create required files if they do not exist
touch "$JOB_QUEUE"
touch "$COMPLETED_JOBS"
touch "$SCHEDULER_LOG"

# ----------------------------------------------------------
# Function: Log scheduler activity
# ----------------------------------------------------------
log_event() {
    local student_id="$1"
    local job_name="$2"
    local scheduling_type="$3"
    local event="$4"

    echo "$(date '+%Y-%m-%d %H:%M:%S') | Student ID: $student_id | Job: $job_name | Scheduling: $scheduling_type | Event: $event" >> "$SCHEDULER_LOG"
}

# ----------------------------------------------------------
# Function: Display main menu
# ----------------------------------------------------------
show_menu() {
    echo
    echo "======================================================"
    echo "       UNIVERSITY RESEARCH CLUSTER SCHEDULER"
    echo "======================================================"
    echo "1. View Pending Jobs"
    echo "2. Submit a Job Request"
    echo "3. Process Queue - Round Robin"
    echo "4. Process Queue - Priority Scheduling"
    echo "5. View Completed Jobs"
    echo "6. View Scheduler Log"
    echo "7. Exit"
    echo "======================================================"
}

# ----------------------------------------------------------
# Function: View pending jobs
# ----------------------------------------------------------
view_pending_jobs() {
    echo
    echo "======================================================"
    echo "           PENDING JOBS QUEUE"
    echo "======================================================"
    
    if [[ ! -s "$JOB_QUEUE" ]]; then
        echo "No jobs in the queue."
        return
    fi
    
    echo "Student ID | Job Name                | Est. Time | Priority"
    echo "----------------------------------------------------------"
    
    # Format and display job queue
    while IFS='|' read -r student_id job_name exec_time priority; do
        # Trim whitespace
        student_id=$(echo "$student_id" | xargs)
        job_name=$(echo "$job_name" | xargs)
        exec_time=$(echo "$exec_time" | xargs)
        priority=$(echo "$priority" | xargs)
        
        # Pad job name to 20 characters for alignment
        printf "%-11s | %-23s | %-9s | %s\n" "$student_id" "$job_name" "$exec_time" "$priority"
    done < "$JOB_QUEUE"
    
    echo "----------------------------------------------------------"
    echo "Total pending jobs: $(wc -l < "$JOB_QUEUE")"
}

# ----------------------------------------------------------
# Function: Submit a job
# ----------------------------------------------------------
submit_job() {
    echo
    echo "======================================================"
    echo "           SUBMIT A JOB REQUEST"
    echo "======================================================"
    
    # Get student ID
    read -p "Enter Student ID: " student_id
    if [[ -z "$student_id" ]]; then
        echo "Error: Student ID cannot be empty."
        return
    fi
    
    # Get job name
    read -p "Enter Job Name: " job_name
    if [[ -z "$job_name" ]]; then
        echo "Error: Job name cannot be empty."
        return
    fi
    
    # Get execution time
    read -p "Enter Estimated Execution Time (seconds): " exec_time
    if [[ -z "$exec_time" || ! "$exec_time" =~ ^[0-9]+$ || "$exec_time" -lt 1 ]]; then
        echo "Error: Execution time must be a positive integer."
        return
    fi
    
    # Get priority
    read -p "Enter Priority (1 = highest, 10 = lowest): " priority
    if [[ -z "$priority" || ! "$priority" =~ ^[0-9]+$ || "$priority" -lt 1 || "$priority" -gt 10 ]]; then
        echo "Error: Priority must be between 1 and 10."
        return
    fi
    
    # Add job to queue
    echo "$student_id|$job_name|$exec_time|$priority" >> "$JOB_QUEUE"
    
    echo "Job submitted successfully!"
    log_event "$student_id" "$job_name" "SUBMISSION" "Job submitted to queue"
}

# ----------------------------------------------------------
# Function: Process queue using Round Robin
# ----------------------------------------------------------
process_round_robin() {
    echo
    echo "======================================================"
    echo "     PROCESSING QUEUE - ROUND ROBIN SCHEDULING"
    echo "======================================================"
    
    if [[ ! -s "$JOB_QUEUE" ]]; then
        echo "No jobs in the queue to process."
        return
    fi
    
    # Create temporary queue file for processing
    local temp_queue="temp_queue_$$.txt"
    cp "$JOB_QUEUE" "$temp_queue"
    
    # Clear the original queue (will be updated as jobs complete)
    > "$JOB_QUEUE"
    
    echo "Time Quantum: $TIME_QUANTUM seconds"
    echo "------------------------------------------"
    
    local job_count=0
    local total_jobs=$(wc -l < "$temp_queue")
    
    # Process until all jobs are complete
    while [[ -s "$temp_queue" ]]; do
        # Read first job from temp queue
        local job_line=$(head -n 1 "$temp_queue")
        # Remove first line from temp queue
        sed -i '1d' "$temp_queue"
        
        # Parse job fields
        IFS='|' read -r student_id job_name remaining_time priority <<< "$job_line"
        
        # Trim whitespace
        student_id=$(echo "$student_id" | xargs)
        job_name=$(echo "$job_name" | xargs)
        remaining_time=$(echo "$remaining_time" | xargs)
        priority=$(echo "$priority" | xargs)
        
        echo "Processing job: $job_name (Student: $student_id)"
        echo "  Remaining time: $remaining_time seconds"
        
        # Determine execution time for this quantum
        if [[ "$remaining_time" -gt "$TIME_QUANTUM" ]]; then
            # Job requires more time
            echo "  Executed for: $TIME_QUANTUM seconds"
            new_remaining=$((remaining_time - TIME_QUANTUM))
            log_event "$student_id" "$job_name" "RR" "Executed $TIME_QUANTUM sec, remaining $new_remaining sec"
            
            # Add job back to temp queue
            echo "$student_id|$job_name|$new_remaining|$priority" >> "$temp_queue"
        else
            # Job completes in this quantum
            echo "  Executed for: $remaining_time seconds (COMPLETED)"
            log_event "$student_id" "$job_name" "RR" "Job completed"
            
            # Move to completed jobs
            echo "$student_id|$job_name|$remaining_time|$priority|$(date '+%Y-%m-%d %H:%M:%S')" >> "$COMPLETED_JOBS"
            job_count=$((job_count + 1))
        fi
        echo "------------------------------------------"
    done
    
    # Clean up temp file
    rm -f "$temp_queue"
    
    echo "Round Robin processing complete."
    echo "Jobs completed: $job_count"
}

# ----------------------------------------------------------
# Function: Process queue using Priority Scheduling
# ----------------------------------------------------------
process_priority() {
    echo
    echo "======================================================"
    echo "    PROCESSING QUEUE - PRIORITY SCHEDULING"
    echo "======================================================"
    
    if [[ ! -s "$JOB_QUEUE" ]]; then
        echo "No jobs in the queue to process."
        return
    fi
    
    # Sort jobs by priority (1 is highest, so ascending order)
    # Create sorted queue
    local sorted_queue="sorted_queue_$$.txt"
    sort -t'|' -k4n "$JOB_QUEUE" > "$sorted_queue"
    
    # Clear original queue
    > "$JOB_QUEUE"
    
    echo "Priority Order (1 = Highest, 10 = Lowest)"
    echo "------------------------------------------"
    
    local job_count=0
    
    # Process each job in priority order
    while IFS= read -r job_line; do
        # Parse job fields
        IFS='|' read -r student_id job_name exec_time priority <<< "$job_line"
        
        # Trim whitespace
        student_id=$(echo "$student_id" | xargs)
        job_name=$(echo "$job_name" | xargs)
        exec_time=$(echo "$exec_time" | xargs)
        priority=$(echo "$priority" | xargs)
        
        echo "Processing job: $job_name (Student: $student_id)"
        echo "  Estimated time: $exec_time seconds"
        echo "  Priority: $priority"
        echo "  Status: COMPLETED"
        echo "------------------------------------------"
        
        # Log completion
        log_event "$student_id" "$job_name" "PRIORITY" "Job completed (Priority: $priority)"
        
        # Move to completed jobs
        echo "$student_id|$job_name|$exec_time|$priority|$(date '+%Y-%m-%d %H:%M:%S')" >> "$COMPLETED_JOBS"
        job_count=$((job_count + 1))
        
    done < "$sorted_queue"
    
    # Clean up temp file
    rm -f "$sorted_queue"
    
    echo "Priority Scheduling complete."
    echo "Jobs processed: $job_count"
}

# ----------------------------------------------------------
# Function: View completed jobs
# ----------------------------------------------------------
view_completed_jobs() {
    echo
    echo "======================================================"
    echo "           COMPLETED JOBS"
    echo "======================================================"
    
    if [[ ! -s "$COMPLETED_JOBS" ]]; then
        echo "No completed jobs."
        return
    fi
    
    echo "Student ID | Job Name                | Time | Priority | Completion Time"
    echo "----------------------------------------------------------------------"
    
    # Format and display completed jobs
    while IFS='|' read -r student_id job_name exec_time priority completion_time; do
        # Trim whitespace
        student_id=$(echo "$student_id" | xargs)
        job_name=$(echo "$job_name" | xargs)
        exec_time=$(echo "$exec_time" | xargs)
        priority=$(echo "$priority" | xargs)
        completion_time=$(echo "$completion_time" | xargs)
        
        printf "%-11s | %-23s | %-5s | %-8s | %s\n" "$student_id" "$job_name" "$exec_time" "$priority" "$completion_time"
    done < "$COMPLETED_JOBS"
    
    echo "----------------------------------------------------------------------"
    echo "Total completed jobs: $(wc -l < "$COMPLETED_JOBS")"
}

# ----------------------------------------------------------
# Function: View scheduler log
# ----------------------------------------------------------
view_scheduler_log() {
    echo
    echo "======================================================"
    echo "           SCHEDULER ACTIVITY LOG"
    echo "======================================================"
    
    if [[ ! -s "$SCHEDULER_LOG" ]]; then
        echo "No log entries found."
        return
    fi
    
    tail -n 30 "$SCHEDULER_LOG"
    echo
    echo "Total log entries: $(wc -l < "$SCHEDULER_LOG")"
}

# ----------------------------------------------------------
# Main program
# ----------------------------------------------------------
while true
do
    show_menu
    
    read -p "Enter your choice: " choice
    
    case "$choice" in
        1)
            view_pending_jobs
            ;;
        2)
            submit_job
            ;;
        3)
            process_round_robin
            ;;
        4)
            process_priority
            ;;
        5)
            view_completed_jobs
            ;;
        6)
            view_scheduler_log
            ;;
        7)
            read -p "Are you sure you want to exit? (Y/N): " confirm
            if [[ "$confirm" == "Y" || "$confirm" == "y" ]]; then
                echo "Exiting Research Cluster Scheduler..."
                exit 0
            else
                echo "Exit cancelled."
            fi
            ;;
        *)
            echo "Invalid choice. Please enter a number from 1 to 7."
            ;;
    esac
done
