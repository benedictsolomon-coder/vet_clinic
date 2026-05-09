**Overview and Purpose**

The Veterinary Clinic Appointment Management System is a console-based application written in C language that helps clinic staff manage patient queues and appointments efficiently. When a pet owner visits or contacts the clinic, a staff member registers them by entering the owner's name, pet name, pet type, severity level (Critical, Urgent, Moderate, or Routine), and a preferred appointment date and time. The system then automatically computes a priority score for each patient and places them in a queue, ensuring that the most critical cases are always attended first while still being fair to patients who have been waiting a long time.

The priority score is calculated using the formula priorityScore = severityScore - waitScore + apptScore. The severity score assigns a base value of 0 for Critical, 10 for Urgent, 20 for Moderate, and 30 for Routine. The wait score, which is subtracted, rewards patients who have been waiting longer by lowering their score, making them rise in the queue over time. A lower total score means higher priority, so the patient at the top of the queue is always the one who needs to be seen most urgently. The system supports up to 100 active patients in the queue at a time and keeps a permanent record of every patient ever registered, including those already served or appointments that had been cancelled.

**Data Structures and Algorithms**

The system uses three core data structures together with bubble sort and circular linear search. A sorted linked list serves as the live patient queue, kept in ascending order by priority score at all times so the most urgent patient is always at the head — adding a patient inserts them into the correct sorted position automatically, serving simply removes the head with no searching required, and cancelling an appointment walks the list to remove a specific node by ID. An array of structs keeps a permanent record of every registered patient ever added, supporting addition, ID-based lookup, and removal by shifting remaining elements left when a record is cancelled or served. An array-based stack enables undo functionality by saving a copy of each patient as they are added, so if the last addition needs to be reversed it can be popped off and cleanly removed from both the queue and the records array.

For display purposes, bubble sort is applied to a temporary copy of the queue — never the real one — so patients can be shown in sorted order by priority score without disturbing the actual linked list. Circular linear search handles name-based lookups by scanning the records array starting from any index and wrapping around to ensure all matches are found, even when duplicate names exist across the full record set.

**Features**

The system offers ten menu options which is numbered from 1-0: 
1. Add patient to queue
2. Serve next patient
3. View queue
4. Search Patient
5. View patient records (BST)
6. View history (Served Patients)
7. Undo Last Add
8. Cancel Appointment
9. View Cancelled Apointments
0. Exit

**Compilationa and Running**

This is how you compile and run it in terminal or your command prompt (CMD) for Windows, macOS, and Linux.

"gcc -o vet_clinic vet_clinic.c" type this on your terminal to compile it and for you to be able to run the program.

"./vet_clinic" use this command to run the program in macOS, and Linux.

"vet_clinic.exe" use this command to run the program in Windows.


#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <ctype.h>
#include <time.h>


// CONSTANTS  —  change these if you want different limits
#define MAX_PATIENTS 100   // max patients the queue can hold
#define MAX_RECORDS  200   // max records the array can hold
#define NAME_LEN     50    // max characters for any name

// Severity levels — lower number = more urgent
#define SEV_CRITICAL 1
#define SEV_URGENT   2
#define SEV_MODERATE 3
#define SEV_ROUTINE  4


/* ============================================================
   STRUCTS  —  the blueprint for our data
============================================================ */

// One patient record. Add or remove fields here.
typedef struct {
    int  id;
    char ownerName[NAME_LEN];
    char petName[NAME_LEN];
    char petType[NAME_LEN];
    int  severity;           // 1=Critical, 2=Urgent, 3=Moderate, 4=Routine
    int  appointmentDate;    // stored as YYYYMMDD, e.g. 20260115 — Moderate/Routine only
    int  appointmentTime;    // stored as HHMM, e.g. 0930 (24h) — Moderate/Routine only
    int  waitMinutes;        // computed at serve time — how long they waited
    time_t timeAdded;        // timestamp when patient was added to queue
    int  priorityScore;      // computed — lower score = served first
} Patient;

/* --- Sorted Linked List node (PRIORITY QUEUE) ---
   Always kept sorted by priorityScore (ascending).
   Head = most urgent patient. */
typedef struct QueueNode {
    Patient          patient;
    struct QueueNode *next;
} QueueNode;

typedef struct {
    QueueNode *head;
    int        size;
} SortedList;

/* --- Array of Structs (PATIENT RECORDS) ---
   Stores every patient ever added.
   Circular search scans from any index and wraps around. */
typedef struct {
    Patient data[MAX_RECORDS];
    int     count;
} RecordArray;

/* --- Linked List node (HISTORY) --- */
typedef struct HistoryNode {
    Patient            patient;
    struct HistoryNode *next;
} HistoryNode;

typedef struct {
    HistoryNode *head;
    int          count;
} LinkedList;




/* ============================================================
   GLOBALS  —  our main data structures
   ============================================================ */
SortedList  queue   = { .head = NULL, .size = 0 };
RecordArray records = { .count = 0 };
LinkedList  history = { .head = NULL, .count = 0 };
int         nextId  = 1;   // auto-increment patient ID


/* ============================================================
   SECTION 1: HELPERS
   ============================================================ */

// Returns the text label for a severity number
const char *severityLabel(int s) {
    switch (s) {
        case SEV_CRITICAL: return "CRITICAL";
        case SEV_URGENT:   return "URGENT";
        case SEV_MODERATE: return "MODERATE";
        default:           return "ROUTINE";
    }
}

// Clears leftover characters in the input buffer
#define clearInput() do { int c; while ((c = getchar()) != '\n' && c != EOF); } while(0)

// Prints a divider line
void printDivider(char c, int n) {
    for (int i = 0; i < n; i++) putchar(c);
    putchar('\n');
}

// Converts string to lowercase into dest
void strToLower(char *dest, const char *src) {
    int i = 0;
    while (src[i]) {
        dest[i] = tolower((unsigned char)src[i]);
        i++;
    }
    dest[i] = '\0';
}

// Formats an appointmentDate (YYYYMMDD) into a readable string like "Jan 15, 2026"
void formatDate(char *out, int date) {
    if (date == 0) {
        strcpy(out, "N/A");
        return;
    }
    int year  = date / 10000;
    int month = (date / 100) % 100;
    int day   = date % 100;
    const char *months[] = {
        "Jan","Feb","Mar","Apr","May","Jun",
        "Jul","Aug","Sep","Oct","Nov","Dec"
    };
    if (month < 1 || month > 12) {
        sprintf(out, "%08d", date);
        return;
    }
    sprintf(out, "%s %02d, %04d", months[month - 1], day, year);
}

// Formats an appointmentTime (HHMM) into "HH:MM" string
void formatTime(char *out, int t) {
    if (t == 0) {
        strcpy(out, "N/A  ");
        return;
    }
    int hour   = t / 100;
    int minute = t % 100;
    const char *period = (hour < 12) ? "PM" : "AM";
    int hour12 = hour % 12;
    if (hour12 == 0) hour12 = 12;   // midnight (0) and noon (12) → display as 12
    sprintf(out, "%02d:%02d %s", hour12, minute, period);
}


/* ============================================================
   SECTION 2: PRIORITY FORMULA
   LOWER score = served FIRST.
   Based solely on severity.
   ============================================================ */
int computePriority(int severity) {
    switch (severity) {
        case SEV_CRITICAL: return  0;
        case SEV_URGENT:   return 10;
        case SEV_MODERATE: return 20;
        default:           return 30;   // ROUTINE
    }
}


/* ============================================================
   SECTION 3: SORTED LINKED LIST  (PRIORITY QUEUE)
   Nodes are inserted in sorted order by priorityScore so the
   head is always the most urgent patient.
   Serving simply removes the head — no searching needed.
   ============================================================ */

// Inserts a patient into the correct position in the sorted list
void listEnqueue(Patient p) {
    QueueNode *node = (QueueNode *)malloc(sizeof(QueueNode));
    if (!node) { printf("  [!] Memory error.\n"); return; }
    node->patient = p;
    node->next    = NULL;

// Case 1: list is empty or new node belongs at the front
    if (!queue.head || p.priorityScore < queue.head->patient.priorityScore) {
        node->next = queue.head;
        queue.head = node;
        queue.size++;
        return;
    }

// Case 2: walk until we find the right spot
    QueueNode *cur = queue.head;
    while (cur->next && cur->next->patient.priorityScore <= p.priorityScore)
        cur = cur->next;

node->next = cur->next;
    cur->next  = node;
    queue.size++;
}

// Removes and returns the head (most urgent patient)
Patient listDequeue() {
    QueueNode *tmp    = queue.head;
    Patient    served = tmp->patient;
    queue.head        = queue.head->next;
    queue.size--;
    free(tmp);
    return served;
}

// Removes a specific patient by ID from the sorted list (used by undo)
int listRemoveById(int id) {
    QueueNode **cur = &queue.head;   // pointer to the pointer we need to update
    while (*cur) {
        if ((*cur)->patient.id == id) {
            QueueNode *tmp = *cur;
            *cur = tmp->next;        // bypass the node
            free(tmp);
            queue.size--;
            return 1;
        }
        cur = &(*cur)->next;
    }
    return 0;
}

// Frees the entire queue list (EXIT)
void listFree() {
    QueueNode *cur = queue.head;
    while (cur) {
        QueueNode *tmp = cur;
        cur = cur->next;
        free(tmp);
    }
    queue.head = NULL;
    queue.size = 0;
}


/* ============================================================
   SECTION 4: ARRAY OF STRUCTS  (PATIENT RECORDS)
   Every patient added is stored here permanently.
   Circular Linear Search starts from a given index and wraps
   around the array so ALL matches are found even if names repeat.
   ============================================================ */

// Adds a patient record to the array
void recordAdd(Patient p) {
    if (records.count >= MAX_RECORDS) {
        printf("  [!] Records array is full.\n");
        return;
    }
    records.data[records.count++] = p;
}

// Removes a record by ID (used by undo) — shifts remaining records left
void recordRemoveById(int id) {
    for (int i = 0; i < records.count; i++) {
        if (records.data[i].id == id) {
            for (int j = i; j < records.count - 1; j++)
                records.data[j] = records.data[j + 1];
            records.count--;
            return;
        }
    }
}

// Linear search by owner name — prints ALL matches including duplicate names
void searchByName(const char *name) {
    char lname[NAME_LEN], lowner[NAME_LEN];
    strToLower(lname, name);

int found = 0;
    for (int i = 0; i < records.count; i++) {
        strToLower(lowner, records.data[i].ownerName);
        if (strstr(lowner, lname)) {
            Patient *p = &records.data[i];
            char dateStr[20], timeStr[10];
            formatDate(dateStr, p->appointmentDate);
            formatTime(timeStr, p->appointmentTime);
            printf("  ID: %-4d | Owner: %-20s | Pet: %-15s (%s)\n",
                   p->id, p->ownerName, p->petName, p->petType);
            printf("         | Severity: %-10s | Score: %d\n",
                   severityLabel(p->severity), p->priorityScore);
            if (p->severity == SEV_MODERATE || p->severity == SEV_ROUTINE)
                printf("         | Appt: %s at %s\n", dateStr, timeStr);
            printDivider('-', 75);
            found++;
        }
    }

if (found == 0)
        printf("  [!] No patient with owner name containing '%s' found.\n", name);
    else
        printf("  %d result(s) found.\n", found);
}

// Searches records by exact ID — returns index or -1
int recordSearchById(int id) {
    for (int i = 0; i < records.count; i++) {
        if (records.data[i].id == id) return i;
    }
    return -1;
}

// Prints all records in the array
void recordPrintAll() {
    if (records.count == 0) {
        printf("  [No records yet]\n");
        return;
    }
    printf("  %-5s %-20s %-15s %-10s %-10s %-13s %-6s %-6s %-5s\n",
           "ID", "Owner", "Pet", "Type", "Severity", "Date", "Time", "Wait", "Score");
    printDivider('-', 95);
    for (int i = 0; i < records.count; i++) {
        Patient *p = &records.data[i];
        char dateStr[20], timeStr[10];
        formatDate(dateStr, p->appointmentDate);
        formatTime(timeStr, p->appointmentTime);
        printf("  %-5d %-20s %-15s %-10s %-10s %-13s %-6s %-6d %-5d\n",
               p->id, p->ownerName, p->petName, p->petType,
               severityLabel(p->severity), dateStr, timeStr,
               p->waitMinutes, p->priorityScore);
    }
    printf("\n  Total records: %d\n", records.count);
}


/* ============================================================
   SECTION 5: HISTORY LINKED LIST
   New served patients are added to the FRONT so the most
   recently served always appears first.
   ============================================================ */

void historyPrepend(Patient p) {
    HistoryNode *node = (HistoryNode *)malloc(sizeof(HistoryNode));
    if (!node) { printf("  [!] Memory error.\n"); return; }
    node->patient = p;
    node->next    = history.head;
    history.head  = node;
    history.count++;
}

void historyPrint() {
    if (!history.head) {
        printf("\n  [No patients served yet]\n");
        return;
    }
    printf("\n  %-4s %-5s %-20s %-15s %-10s %-10s %-5s\n",
           "No.", "ID", "Owner", "Pet", "Type", "Severity", "Score");
    printDivider('-', 75);

HistoryNode *cur = history.head;
    int num = 1;
    while (cur) {
        Patient *p = &cur->patient;
        printf("  %-4d %-5d %-20s %-15s %-10s %-10s %-5d\n",
               num++, p->id, p->ownerName, p->petName,
               p->petType, severityLabel(p->severity), p->priorityScore);
        cur = cur->next;
    }
    printf("\n  Total served: %d\n", history.count);
}

void historyFree() {
    HistoryNode *cur = history.head;
    while (cur) {
        HistoryNode *tmp = cur;
        cur = cur->next;
        free(tmp);
    }
    history.head  = NULL;
    history.count = 0;
}


/* ============================================================
   SECTION 6: TIME SLOT CONFLICT CHECKER
   Checks the QUEUE only — once a patient is served or cancelled,
   their slot becomes available again for new appointments.
   ============================================================ */
// Returns 1 if the date+time slot is already taken, 0 if free
int isTimeSlotTaken(int date, int time) {
    QueueNode *cur = queue.head;
    while (cur) {
        if (cur->patient.appointmentDate == date &&
            cur->patient.appointmentTime == time) {
            printf("\n  [!] This time slot is already booked by:\n");
            printf("      Owner : %s\n", cur->patient.ownerName);
            printf("      Pet   : %s (ID: %d)\n", cur->patient.petName, cur->patient.id);
            printf("      Slot will be free once that patient is served or cancelled.\n");
            return 1;   // slot is taken
        }
        cur = cur->next;
    }
    return 0;   // slot is free
}


/* ============================================================
   SECTION 7: BUBBLE SORT
   Sorts a temporary array copy of the queue by priority score.
   The actual sorted linked list order is NOT changed.
   ============================================================ */
void bubbleSort(Patient *arr, int n) {
    for (int i = 0; i < n - 1; i++) {
        int swapped = 0;
        for (int j = 0; j < n - i - 1; j++) {
            if (arr[j].priorityScore > arr[j + 1].priorityScore) {
                Patient tmp  = arr[j];
                arr[j]       = arr[j + 1];
                arr[j + 1]   = tmp;
                swapped = 1;
            }
        }
        if (!swapped) break;   // already sorted — stop early
    }
}


/* ============================================================
   SECTION 8: DISPLAY FUNCTIONS
   ============================================================ */

void printHeader() {
    printf("\n");
    printDivider('=', 56);
    printf("   /\\_/\\                     __                    __           \n");
    printf("  ( o.o )         ___  ___  / /_   \\ \\  / /  ___  / /_         \n");
    printf("   > ^ <        / __ \\/ _ \\/ __/    \\ \\/ / /  _ \\/  __/             \n");
    printf("  /|   |\\      / /_/ /  __/ /_       \\  / /  __/  /_              \n");
    printf(" (_|   |_)    / .___/\\___/\\__/        \\/  \\___/\\___/                    \n");
    printf("             /_/                                                        \n");
    printDivider('=', 56);
    printf("   VETCARE CLINIC - APPOINTMENT MANAGEMENT SYSTEM\n");
    printDivider('=', 56);
}

void printMainMenu() {
    printf("\n  +--------------------------------------+\n");
    printf("  |  1. Add Patient to Queue             |\n");
    printf("  |  2. Serve Next Patient               |\n");
    printf("  |  3. View Queue                       |\n");
    printf("  |  4. Search Patient                   |\n");
    printf("  |  5. View All Records                 |\n");
    printf("  |  6. View Serve History               |\n");
    printf("  |  7. Cancel Appointment               |\n");
    printf("  |  8. Exit                             |\n");
    printf("  +--------------------------------------+\n");
    printf("  Choice: ");
}

// Copies queue into a temp array, bubble sorts it by priority, then displays
void displayQueue() {
    if (queue.size == 0) {
        printf("\n  [Queue is empty]\n");
        return;
    }

// Copy linked list nodes into a temp array
    Patient   copy[MAX_PATIENTS];
    int       n   = 0;
    QueueNode *cur = queue.head;
    while (cur && n < MAX_PATIENTS) {
        copy[n++] = cur->patient;
        cur = cur->next;
    }

// Bubble sort the copy by priority score
    bubbleSort(copy, n);

    printf("\n  Sorted by: Priority Score\n");
    printf("  %-4s %-5s %-20s %-15s %-10s %-10s %-13s %-6s %-6s %-5s\n",
           "Rank", "ID", "Owner", "Pet", "Type",
           "Severity", "Date", "Time", "Wait", "Priority");
    printDivider('-', 110);

for (int i = 0; i < n; i++) {
        Patient *p = &copy[i];
        char dateStr[20], timeStr[10];
        formatDate(dateStr, p->appointmentDate);
        formatTime(timeStr, p->appointmentTime);
        int wait = (int)(difftime(time(NULL), p->timeAdded) / 60);
        printf("  %-4d %-5d %-20s %-15s %-10s %-10s %-13s %-6s %-6d %-5d\n",
               i + 1, p->id, p->ownerName, p->petName, p->petType,
               severityLabel(p->severity), dateStr, timeStr,
               wait, p->priorityScore);
    }
    printf("\n  Patients in queue: %d\n", queue.size);
}


/* ============================================================
   SECTION 9: INPUT FUNCTIONS
   ============================================================ */

int getSeverity() {
    int s;
    printf("  Severity (1 = Critical, 2 = Urgent, 3 = Moderate, 4 = Routine): ");
    while (scanf("%d", &s) != 1 || s < 1 || s > 4) {
        printf("  Invalid. Enter 1 to 4: ");
        clearInput();
    }
    clearInput();
    return s;
}

// Returns a validated date as an integer YYYYMMDD
int getAppointmentDate() {
    int year, month, day;
    int daysInMonth[] = {0, 31, 28, 31, 30, 31, 30, 31, 31, 30, 31, 30, 31};

    while (1) {
        printf("  Appointment Date (YYYY MM DD): ");
        if (scanf("%d %d %d", &year, &month, &day) != 3) {
            printf("  Invalid. Enter year, month, and day separated by spaces.\n");
            clearInput();
            continue;
        }
        clearInput();

        // Basic range checks
        if (year < 1900 || year > 9999) {
            printf("  Invalid year. Try again.\n");
            continue;
        }
        if (month < 1 || month > 12) {
            printf("  Invalid month (1-12). Try again.\n");
            continue;
        }

        // Leap year adjustment for February
        int maxDay = daysInMonth[month];
        if (month == 2 && ((year % 4 == 0 && year % 100 != 0) || year % 400 == 0))
            maxDay = 29;

        if (day < 1 || day > maxDay) {
            printf("  Invalid day for the given month. Try again.\n");
            continue;
        }

        return year * 10000 + month * 100 + day;   // e.g. 20260115
    }
}

int getAppointmentTime() {
    int h, m;
    char period[5];

    while (1) {
        printf("  Appointment Time (HH MM AM/PM, e.g. 09 30 AM): ");
        if (scanf("%d %d %4s", &h, &m, period) != 3) {
            printf("  Invalid. Try again.\n");
            clearInput();
            continue;
        }
        clearInput();

        // Validate
        if (h < 1 || h > 12 || m < 0 || m > 59) {
            printf("  Invalid time. Hour must be 1-12, minute 0-59.\n");
            continue;
        }

        // Convert to uppercase for comparison
        for (int i = 0; period[i]; i++)
            period[i] = toupper((unsigned char)period[i]);

        if (strcmp(period, "AM") != 0 && strcmp(period, "PM") != 0) {
            printf("  Invalid period. Enter AM or PM.\n");
            continue;
        }

        // Convert to 24-hour HHMM
        int hour24 = h;
        if (strcmp(period, "PM") == 0 && h == 12) hour24 = 0;   // 12 AM = midnight
        if (strcmp(period, "AM") == 0 && h != 12) hour24 = h + 12; // PM shift

        return hour24 * 100 + m;   // stored as 24h internally
    }
}


/* ============================================================
   SECTION 10: MENU ACTIONS
   ============================================================ */

// Collects patient info, adds to sorted list + records array + stack
void addPatient() {
    if (queue.size >= MAX_PATIENTS) {
        printf("\n  [!] Queue is full.\n");
        return;
    }

Patient p;
    p.id = nextId++;

printf("\n--- Add New Patient (ID: %d) ---\n", p.id);

printf("  Owner Name : ");
    fgets(p.ownerName, NAME_LEN, stdin);
    p.ownerName[strcspn(p.ownerName, "\n")] = '\0';
    if (strlen(p.ownerName) == 0) strncpy(p.ownerName, "Unknown", NAME_LEN);

printf("  Pet Type   : ");
    fgets(p.petType, NAME_LEN, stdin);
    p.petType[strcspn(p.petType, "\n")] = '\0';
    if (strlen(p.petType) == 0) strncpy(p.petType, "Unknown", NAME_LEN);

printf("  Pet Name   : ");
    fgets(p.petName, NAME_LEN, stdin);
    p.petName[strcspn(p.petName, "\n")] = '\0';
    if (strlen(p.petName) == 0) strncpy(p.petName, "Unknown", NAME_LEN);

p.severity = getSeverity();

// Appointment date and time only for Moderate and Routine
    if (p.severity == SEV_MODERATE || p.severity == SEV_ROUTINE) {
        int chosenDate, chosenTime;
        while (1) {
            chosenDate = getAppointmentDate();
            chosenTime = getAppointmentTime();
            if (isTimeSlotTaken(chosenDate, chosenTime)) {
                printf("      Please choose a different date or time.\n\n");
            } else {
                break;   // slot is free, proceed
            }
        }
        p.appointmentDate = chosenDate;
        p.appointmentTime = chosenTime;
    } else {
        p.appointmentDate = 0;   // not applicable for Critical / Urgent
        p.appointmentTime = 0;
    }

p.waitMinutes = 0;          // will be computed when served
    p.timeAdded   = time(NULL); // record the current time

p.priorityScore = computePriority(p.severity);

listEnqueue(p);   // insert into sorted linked list in correct position
    recordAdd(p);     // store in records array

    char dateStr[20], timeStr[10];
    formatDate(dateStr, p.appointmentDate);
    formatTime(timeStr, p.appointmentTime);

    printf("\n  [+] %s's pet %s added to queue.\n", p.ownerName, p.petName);
    printf("      Score: %d | Severity: %s\n", p.priorityScore, severityLabel(p.severity));
    if (p.severity == SEV_MODERATE || p.severity == SEV_ROUTINE)
        printf("      Appointment: %s at %s\n", dateStr, timeStr);
}

// Removes the head of the sorted list (most urgent) and adds to history
void serveNext() {
    if (queue.size == 0) {
        printf("\n  [!] No patients in queue.\n");
        return;
    }

Patient served = listDequeue();
    served.waitMinutes = (int)(difftime(time(NULL), served.timeAdded) / 60);
    historyPrepend(served);

    char dateStr[20], timeStr[10];
    formatDate(dateStr, served.appointmentDate);
    formatTime(timeStr, served.appointmentTime);

printf("\n");
    printDivider('*', 50);
    printf("  NOW SERVING\n");
    printDivider('*', 50);
    printf("  Patient ID    : %d\n",      served.id);
    printf("  Owner         : %s\n",      served.ownerName);
    printf("  Pet           : %s (%s)\n", served.petName, served.petType);
    printf("  Severity      : %s\n",      severityLabel(served.severity));
    if (served.severity == SEV_MODERATE || served.severity == SEV_ROUTINE)
        printf("  Appointment   : %s at %s\n", dateStr, timeStr);
    printf("  Wait Time     : %d min\n",  served.waitMinutes);
    printf("  Priority Score: %d\n",      served.priorityScore);
    printDivider('*', 50);
    printf("  Remaining in queue: %d\n",  queue.size);
}

// Search by ID (direct scan) or by owner name using circular search
void searchPatient() {
    printf("\n--- Search Patient ---\n");
    printf("  1. Search by ID\n");
    printf("  2. Search by Owner Name\n");
    printf("  Choice: ");

int choice;
    if (scanf("%d", &choice) != 1) { clearInput(); return; }
    clearInput();

if (choice == 1) {
        int id;
        printf("  Enter Patient ID: ");
        if (scanf("%d", &id) != 1) { clearInput(); return; }
        clearInput();

int idx = recordSearchById(id);
        if (idx >= 0) {
            Patient *p = &records.data[idx];
            char dateStr[20], timeStr[10];
            formatDate(dateStr, p->appointmentDate);
            formatTime(timeStr, p->appointmentTime);
            printf("\n  [Record Search] Found:\n");
            printf("  ID: %d | Owner: %s | Pet: %s (%s)\n",
                   p->id, p->ownerName, p->petName, p->petType);
            printf("  Severity: %s | Score: %d\n",
                   severityLabel(p->severity), p->priorityScore);
            if (p->severity == SEV_MODERATE || p->severity == SEV_ROUTINE)
                printf("  Appointment: %s at %s\n", dateStr, timeStr);

// Check if still in queue by walking the list
            int inQueue = 0;
            QueueNode *cur = queue.head;
            while (cur) {
                if (cur->patient.id == id) { inQueue = 1; break; }
                cur = cur->next;
            }
            printf("  Status: %s\n", inQueue ? "Still in queue" : "Already served or removed");
        } else {
            printf("\n  [!] No patient with ID %d found.\n", id);
        }

} else if (choice == 2) {
        char name[NAME_LEN];
        printf("  Enter Owner Name: ");
        fgets(name, NAME_LEN, stdin);
        name[strcspn(name, "\n")] = '\0';

printf("\n  [Name Search] Results for '%s':\n", name);
        printDivider('-', 75);
        searchByName(name);

} else {
        printf("  Invalid choice.\n");
    }
}


// Cancels a patient's appointment by ID — removes from queue and records
void cancelAppointment() {
    printf("\n--- Cancel Appointment ---\n");
    printf("  Enter Patient ID to cancel: ");

int id;
    if (scanf("%d", &id) != 1) { clearInput(); return; }
    clearInput();

int recIdx = recordSearchById(id);
    if (recIdx < 0) {
        printf("\n  [!] No patient with ID %d found.\n", id);
        return;
    }

Patient p = records.data[recIdx];

// Check if still in queue
    int inQueue = 0;
    QueueNode *cur = queue.head;
    while (cur) {
        if (cur->patient.id == id) { inQueue = 1; break; }
        cur = cur->next;
    }

if (!inQueue) {
        printf("\n  [!] Patient ID %d (%s / %s) has already been served or removed.\n",
               id, p.ownerName, p.petName);
        return;
    }

    char dateStr[20], timeStr[10];
    formatDate(dateStr, p.appointmentDate);
    formatTime(timeStr, p.appointmentTime);

// Show info and confirm
    printf("\n  Patient found:\n");
    printf("  ID: %d | Owner: %s | Pet: %s (%s) | Severity: %s\n",
           p.id, p.ownerName, p.petName, p.petType, severityLabel(p.severity));
    if (p.severity == SEV_MODERATE || p.severity == SEV_ROUTINE)
        printf("  Appointment: %s at %s\n", dateStr, timeStr);
    printf("\n  Confirm cancellation? (1 = Yes, 0 = No): ");

int confirm;
    if (scanf("%d", &confirm) != 1) { clearInput(); return; }
    clearInput();

if (confirm != 1) {
        printf("\n  Cancellation aborted.\n");
        return;
    }

listRemoveById(id);
    recordRemoveById(id);

printf("\n  [-] Appointment cancelled for: %s (Pet: %s, ID: %d)\n",
           p.ownerName, p.petName, p.id);
    printf("      Remaining patients in queue: %d\n", queue.size);
}


/* ============================================================
   MAIN  —  the program loop
   ============================================================ */
int main() {
    printHeader();

int choice;
    while (1) {
        printMainMenu();

if (scanf("%d", &choice) != 1) { clearInput(); continue; }
        clearInput();

switch (choice) {
            case 1:
                addPatient();
                break;

case 2:
                serveNext();
                break;

case 3:
                printf("\n--- Current Queue ---");
                displayQueue();
                break;

case 4:
                searchPatient();
                break;

case 5:
                printf("\n--- All Patient Records ---\n");
                recordPrintAll();
                break;

case 6:
                printf("\n--- Serve History (Most Recent First) ---");
                historyPrint();
                break;

case 7:
                cancelAppointment();
                break;

case 8:
                printf("\n  Goodbye! Stay healthy, pets!\n\n");
                listFree();
                historyFree();
                return 0;

default:
                printf("\n  Invalid choice. Enter 1-8.\n");
        }
    }
}
