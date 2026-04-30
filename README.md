**Overview and Purpose**

The Veterinary Clinic Appointment Management System is a console-based application written in C language that helps clinic staff manage patient queues and appointments efficiently. When a pet owner visits or contacts the clinic, a staff member registers them by entering the owner's name, pet name, pet type, severity level (Critical, Urgent, Moderate, or Routine), and a preferred appointment date and time. The system then automatically computes a priority score for each patient and places them in a queue, ensuring that the most critical cases are always attended first while still being fair to patients who have been waiting a long time.

The priority score is calculated using the formula priorityScore = severityScore - waitScore + apptScore. The severity score assigns a base value of 0 for Critical, 10 for Urgent, 20 for Moderate, and 30 for Routine. The wait score, which is subtracted, rewards patients who have been waiting longer by lowering their score, making them rise in the queue over time. A lower total score means higher priority, so the patient at the top of the queue is always the one who needs to be seen most urgently. The system supports up to 100 active patients in the queue at a time and keeps a permanent record of every patient ever registered, including those already served or appointments that had been cancelled.

**Data Structures and Algorithms**

The system uses four core data structures together with bubble sort and linear search. The min-heap serves as the live patient queue, always keeping the most urgent patient at the top so the system can instantly find who to serve next, it also supports adding patients, serving them, cancelling appointments, and undoing additions. Two singly linked lists maintain the served patient history and the cancelled appointments section, where each new entry is simply attached to the front of the list so the most recent record always appears first. An array-based stack enables undo functionality by saving a copy of each patient as they are added, so if the last addition needs to be reversed, it can be removed quickly and cleanly. A binary search tree (BST) focused on patient ID keeps a permanent record of every registered patient, organized in a way that makes looking up, adding, and removing records fast without scanning the entire list. For display purposes, bubble sort is applied to a temporary copy of the queue, never the real one to offer five sort modes (priority, severity, wait time, appointment time, and name), while linear search handles name-based lookups, appointment slot conflict checks, and ID searches when a quick BST lookup is not enough.

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

========== Source code ==========

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <ctype.h>
#include <time.h>

#define MAX_PATIENTS  100
#define NAME_LEN      50
#define STACK_SIZE    100

#define SEV_CRITICAL  1
#define SEV_URGENT    2
#define SEV_MODERATE  3
#define SEV_ROUTINE   4

#define SORT_PRIORITY  0
#define SORT_SEVERITY  1
#define SORT_WAIT      2
#define SORT_APPT      3
#define SORT_NAME      4

typedef struct {
    char   dateStr[30];
    char   timeStr[20];
    char   dayName[15];
    int    hhmm;
} AppointmentInfo;

typedef struct {
    int             id;
    char            ownerName[NAME_LEN];
    char            petName[NAME_LEN];
    char            petType[NAME_LEN];
    int             severity;
    AppointmentInfo appt;
    int             waitMinutes;
    time_t          enrolledAt;
    int             priorityScore;
} Patient;

typedef struct {
    Patient data[MAX_PATIENTS];
    int     size;
} MinHeap;

typedef struct HistoryNode {
    Patient             patient;
    struct HistoryNode *next;
} HistoryNode;

typedef struct {
    HistoryNode *head;
    int          count;
} LinkedList;

typedef struct {
    Patient data[STACK_SIZE];
    int     top;
} Stack;

typedef struct BSTNode {
    Patient         patient;
    struct BSTNode *left;
    struct BSTNode *right;
} BSTNode;

MinHeap    heap      = { .size = 0 };
LinkedList history   = { .head = NULL, .count = 0 };
LinkedList cancelled = { .head = NULL, .count = 0 };
Stack      undoStk   = { .top = -1 };
BSTNode   *bstRoot   = NULL;
int        nextId    = 1;

/* ------------------------------------------------------------------ */
/*  Helpers                                                             */
/* ------------------------------------------------------------------ */

int computePriority(int severity, int waitMinutes, int hhmm) {
    int sevScore;
    switch (severity) {
        case SEV_CRITICAL: sevScore =  0; break;
        case SEV_URGENT:   sevScore = 10; break;
        case SEV_MODERATE: sevScore = 20; break;
        default:           sevScore = 30; break;
    }
    int waitScore = waitMinutes / 3;
    if (waitScore > 20) waitScore = 20;
    int apptMins  = (hhmm / 100) * 60 + (hhmm % 100);
    int apptScore = apptMins / 64;
    if (apptScore > 15) apptScore = 15;
    return sevScore - waitScore + apptScore;
}

int getElapsedMinutes(time_t enrolledAt) {
    double diff = difftime(time(NULL), enrolledAt);
    int mins = (int)(diff / 60);
    return mins < 0 ? 0 : mins;
}

const char *severityLabel(int s) {
    switch (s) {
        case SEV_CRITICAL: return "CRITICAL";
        case SEV_URGENT:   return "URGENT";
        case SEV_MODERATE: return "MODERATE";
        default:           return "ROUTINE";
    }
}

int isSlotTaken(const AppointmentInfo *a, int severity) {
    if (severity != SEV_MODERATE && severity != SEV_ROUTINE)
        return -1;
    for (int i = 0; i < heap.size; i++) {
        Patient *p = &heap.data[i];
        if (p->severity != SEV_MODERATE && p->severity != SEV_ROUTINE)
            continue;
        if (strcmp(p->appt.dateStr, a->dateStr) == 0 &&
            p->appt.hhmm == a->hhmm)
            return p->id;
    }
    return -1;
}

void clearInput() {
    int c;
    while ((c = getchar()) != '\n' && c != EOF);
}

void printDivider(char c, int n) {
    for (int i = 0; i < n; i++) putchar(c);
    putchar('\n');
}

void strToLower(char *dest, const char *src) {
    int i = 0;
    while (src[i]) { dest[i] = tolower((unsigned char)src[i]); i++; }
    dest[i] = '\0';
}

void readString(char *buf, int len, const char *prompt) {
    printf("%s", prompt);
    fgets(buf, len, stdin);
    buf[strcspn(buf, "\r\n")] = '\0';
}

void apptToString(const AppointmentInfo *a, char *buf, int bufLen) {
    snprintf(buf, bufLen, "%s  %s", a->dateStr, a->timeStr);
}

/* ------------------------------------------------------------------ */
/*  Appointment input                                                   */
/* ------------------------------------------------------------------ */

static int parseTimeStr(const char *s) {
    int hh, mm;
    char suffix[4] = {0};
    if (sscanf(s, "%d:%d%3s", &hh, &mm, suffix) != 3) return -1;
    if (hh < 1 || hh > 12 || mm < 0 || mm > 59)       return -1;
    for (int i = 0; suffix[i]; i++)
        suffix[i] = (char)tolower((unsigned char)suffix[i]);
    if (strcmp(suffix, "am") != 0 && strcmp(suffix, "pm") != 0) return -1;
    if (strcmp(suffix, "pm") == 0 && hh != 12) hh += 12;
    if (strcmp(suffix, "am") == 0 && hh == 12) hh  = 0;
    return hh * 100 + mm;
}

static int parseDateDay(const char *dateStr, char *dayOut, int dayLen) {
    const char *p = strchr(dateStr, '-');
    if (!p) return 0;
    p = strchr(p + 1, '-');
    if (!p) return 0;
    p++;
    if (*p == '\0') return 0;
    strncpy(dayOut, p, dayLen - 1);
    dayOut[dayLen - 1] = '\0';
    return 1;
}

AppointmentInfo getAppointmentInfo() {
    AppointmentInfo a;
    memset(&a, 0, sizeof(a));

   while (1) {
        readString(a.dateStr, sizeof(a.dateStr),
                   "  Date (YYYY-MM-DD, e.g. 2026-02-28): ");
        if (parseDateDay(a.dateStr, a.dayName, sizeof(a.dayName)))
            break;
        printf("  [!] Invalid format. Use YYYY-MM-DD (e.g. 2026-04-05).\n");
    }

   while (1) {
        readString(a.timeStr, sizeof(a.timeStr),
                   "  Time (e.g. 12:00pm or 9:30am): ");
        int hhmm = parseTimeStr(a.timeStr);
        if (hhmm >= 0) { a.hhmm = hhmm; break; }
        printf("  [!] Invalid format. Use H:MMam or H:MMpm (e.g. 2:00pm).\n");
    }

  return a;
}

/* ------------------------------------------------------------------ */
/*  Min-Heap                                                            */
/* ------------------------------------------------------------------ */

void heapSwap(Patient *a, Patient *b) {
    Patient tmp = *a; *a = *b; *b = tmp;
}

void heapifyUp(int idx) {
    while (idx > 0) {
        int parent = (idx - 1) / 2;
        if (heap.data[parent].priorityScore > heap.data[idx].priorityScore) {
            heapSwap(&heap.data[parent], &heap.data[idx]);
            idx = parent;
        } else break;
    }
}

void heapifyDown(int idx) {
    while (1) {
        int left     = 2 * idx + 1;
        int right    = 2 * idx + 2;
        int smallest = idx;
        if (left  < heap.size && heap.data[left].priorityScore  < heap.data[smallest].priorityScore) smallest = left;
        if (right < heap.size && heap.data[right].priorityScore < heap.data[smallest].priorityScore) smallest = right;
        if (smallest != idx) {
            heapSwap(&heap.data[smallest], &heap.data[idx]);
            idx = smallest;
        } else break;
    }
}

void heapEnqueue(Patient p) {
    if (heap.size >= MAX_PATIENTS) {
        printf("  [!] Queue is full.\n");
        return;
    }
    heap.data[heap.size++] = p;
    heapifyUp(heap.size - 1);
}

Patient heapDequeue() {
    Patient top  = heap.data[0];
    heap.data[0] = heap.data[--heap.size];
    heapifyDown(0);
    return top;
}

/* ------------------------------------------------------------------ */
/*  Linked List (history)                                               */
/* ------------------------------------------------------------------ */

void listPrepend(LinkedList *list, Patient p) {
    HistoryNode *node = (HistoryNode *)malloc(sizeof(HistoryNode));
    if (!node) { printf("  [!] Memory error.\n"); return; }
    node->patient = p;
    node->next    = list->head;
    list->head    = node;
    list->count++;
}

void listPrint() {
    if (!history.head) {
        printf("\n  [No patients served yet]\n");
        return;
    }
    printf("\n  %-4s %-5s %-20s %-15s %-10s %-10s %-5s\n",
           "No.","ID","Owner","Pet","Type","Severity","Score");
    printDivider('-', 80);
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

void listFreeAll(LinkedList *list) {
    HistoryNode *cur = list->head;
    while (cur) {
        HistoryNode *tmp = cur;
        cur = cur->next;
        free(tmp);
    }
    list->head  = NULL;
    list->count = 0;
}

/* ------------------------------------------------------------------ */
/*  Stack (undo)                                                        */
/* ------------------------------------------------------------------ */

void stackPush(Patient p) {
    if (undoStk.top >= STACK_SIZE - 1) {
        printf("  [!] Undo history full — oldest entry discarded.\n");
        for (int i = 0; i < STACK_SIZE - 1; i++)
            undoStk.data[i] = undoStk.data[i + 1];
        undoStk.data[undoStk.top] = p;
    } else {
        undoStk.data[++undoStk.top] = p;
    }
}

int stackPop(Patient *out) {
    if (undoStk.top < 0) return 0;
    *out = undoStk.data[undoStk.top--];
    return 1;
}

int stackIsEmpty() { return undoStk.top < 0; }

/* ------------------------------------------------------------------ */
/*  BST                                                                 */
/* ------------------------------------------------------------------ */

BSTNode *bstNewNode(Patient p) {
    BSTNode *n = (BSTNode *)malloc(sizeof(BSTNode));
    if (!n) { printf("  [!] Memory error.\n"); return NULL; }
    n->patient = p;
    n->left = n->right = NULL;
    return n;
}

BSTNode *bstInsert(BSTNode *root, Patient p) {
    if (!root) return bstNewNode(p);
    if (p.id < root->patient.id)
        root->left  = bstInsert(root->left,  p);
    else if (p.id > root->patient.id)
        root->right = bstInsert(root->right, p);
    else
        root->patient = p;
    return root;
}

BSTNode *bstSearch(BSTNode *root, int id) {
    if (!root || root->patient.id == id) return root;
    if (id < root->patient.id) return bstSearch(root->left,  id);
    else                       return bstSearch(root->right, id);
}

void bstInOrder(BSTNode *root, int *count) {
    if (!root) return;
    bstInOrder(root->left, count);
    Patient *p = &root->patient;
    int liveWait = getElapsedMinutes(p->enrolledAt);
    char apptStr[60];
    apptToString(&p->appt, apptStr, sizeof(apptStr));
    (*count)++;
    printf("  %-5d %-20s %-15s %-10s %-10s %-35s %-6d %-5d\n",
           p->id, p->ownerName, p->petName, p->petType,
           severityLabel(p->severity), apptStr,
           liveWait, p->priorityScore);
    bstInOrder(root->right, count);
}

BSTNode *bstFindMin(BSTNode *root) {
    while (root->left) root = root->left;
    return root;
}

BSTNode *bstDelete(BSTNode *root, int id) {
    if (!root) return NULL;
    if (id < root->patient.id)
        root->left  = bstDelete(root->left,  id);
    else if (id > root->patient.id)
        root->right = bstDelete(root->right, id);
    else {
        if (!root->left) {
            BSTNode *tmp = root->right; free(root); return tmp;
        } else if (!root->right) {
            BSTNode *tmp = root->left;  free(root); return tmp;
        }
        BSTNode *succ   = bstFindMin(root->right);
        root->patient   = succ->patient;
        root->right     = bstDelete(root->right, succ->patient.id);
    }
    return root;
}

void bstFree(BSTNode *root) {
    if (!root) return;
    bstFree(root->left);
    bstFree(root->right);
    free(root);
}

/* ------------------------------------------------------------------ */
/*  Sorting / searching                                                 */
/* ------------------------------------------------------------------ */

int comparePatients(Patient *a, Patient *b, int mode) {
    switch (mode) {
        case SORT_SEVERITY: return a->severity        - b->severity;
        case SORT_WAIT:     return b->waitMinutes     - a->waitMinutes;
        case SORT_APPT:     return a->appt.hhmm       - b->appt.hhmm;
        case SORT_NAME:     return strcmp(a->ownerName, b->ownerName);
        default:            return a->priorityScore   - b->priorityScore;
    }
}

void bubbleSort(Patient *arr, int n, int mode) {
    for (int i = 0; i < n - 1; i++) {
        int swapped = 0;
        for (int j = 0; j < n - i - 1; j++) {
            if (comparePatients(&arr[j], &arr[j + 1], mode) > 0) {
                Patient tmp  = arr[j];
                arr[j]       = arr[j + 1];
                arr[j + 1]   = tmp;
                swapped = 1;
            }
        }
        if (!swapped) break;
    }
}

int linearSearchById(int id) {
    for (int i = 0; i < heap.size; i++)
        if (heap.data[i].id == id) return i;
    return -1;
}

int linearSearchByName(const char *name) {
    char lname[NAME_LEN], lpname[NAME_LEN];
    strToLower(lname, name);
    for (int i = 0; i < heap.size; i++) {
        strToLower(lpname, heap.data[i].ownerName);
        if (strstr(lpname, lname)) return i;
    }
    return -1;
}

/* ------------------------------------------------------------------ */
/*  Display                                                             */
/* ------------------------------------------------------------------ */

void displayQueue(int sortMode) {
    if (heap.size == 0) {
        printf("\n  [Queue is empty]\n");
        return;
    }
    Patient copy[MAX_PATIENTS];
    memcpy(copy, heap.data, heap.size * sizeof(Patient));

   for (int i = 0; i < heap.size; i++)
        copy[i].waitMinutes = getElapsedMinutes(copy[i].enrolledAt);

   bubbleSort(copy, heap.size, sortMode);
    const char *sortLabel[] = {
        "Priority Score","Severity","Wait Time","Appointment Time","Owner Name"
    };
    printf("\n  Sorted by: %s\n", sortLabel[sortMode]);
    printf("  %-4s %-5s %-20s %-15s %-10s %-10s %-35s %-6s %-6s\n",
           "Rank","ID","Owner","Pet","Type","Severity","Appointment","Wait","Score");
    printDivider('-', 115);
    for (int i = 0; i < heap.size; i++) {
        Patient *p = &copy[i];
        char apptStr[60];
        apptToString(&p->appt, apptStr, sizeof(apptStr));
        printf("  %-4d %-5d %-20s %-15s %-10s %-10s %-35s %-6d %-6d\n",
               i + 1, p->id, p->ownerName, p->petName, p->petType,
               severityLabel(p->severity), apptStr,
               p->waitMinutes, p->priorityScore);
    }
    printf("\n  Patients in queue: %d\n", heap.size);
}

/* ------------------------------------------------------------------ */
/*  UI helpers                                                          */
/* ------------------------------------------------------------------ */

int getSeverity() {
    int s;
    printf("  Severity (1=Critical, 2=Urgent, 3=Moderate, 4=Routine): ");
    while (scanf("%d", &s) != 1 || s < 1 || s > 4) {
        printf("  Invalid. Enter 1-4: ");
        clearInput();
    }
    clearInput();
    return s;
}

void printHeader() {
    printf("\n");
    printf("  ================================================================\n");
    printf("   /\\_/\\       ____                                     __\n");
    printf("  ( o.o )     / __ \\____  ____ _     ____  ___  / /_\n");
    printf("   > ^ <     / / / / __ \\/ __ `/    / __ \\/ _ \\/ __/\n");
    printf("  /|   |\\   / /_/ / /_/ / /_/ /    / /_/ /  __/ /_\n");
    printf(" (_|   |_)  \\____/\\____/\\__, /     / .___/\\___/\\__/\n");
    printf("                        /____/     /_/\n");
    printf("  ----------------------------------------------------------------\n");
    printf("         VETERINARY CLINIC -- Appointment Management System\n");
    printf("  ================================================================\n");
    printf("\n");
}

void printMainMenu() {
    printf("\n  +--------------------------------------+\n");
    printf("  |  1. Add Patient to Queue             |\n");
    printf("  |  2. Serve Next Patient               |\n");
    printf("  |  3. View Queue                       |\n");
    printf("  |  4. Search Patient                   |\n");
    printf("  |  5. View Patient Records (BST)       |\n");
    printf("  |  6. View History (Served Patients)   |\n");
    printf("  |  7. Undo Last Add                    |\n");
    printf("  |  8. Cancel Appointment               |\n");
    printf("  |  9. View Cancelled Appointments      |\n");
    printf("  |  0. Exit                             |\n");
    printf("  +--------------------------------------+\n");
    printf("  Choice: ");
}

void printSortMenu() {
    printf("\n  Sort by:\n");
    printf("  0. Priority Score (default)\n");
    printf("  1. Severity\n");
    printf("  2. Wait Time (longest first)\n");
    printf("  3. Appointment Time\n");
    printf("  4. Owner Name (A-Z)\n");
    printf("  Choice [0-4]: ");
}

/* ------------------------------------------------------------------ */
/*  Sunday slot helper                                                  */
/* ------------------------------------------------------------------ */

/*
 * releaseSlot: called when a patient is served, cancelled, or undone.
 * Since we check the heap live in isSlotTaken(), no extra state is needed.
 * We just print a message if they were Routine/Moderate.
 */
static void releaseSlot(const Patient *p) {
    if (p->severity == SEV_MODERATE || p->severity == SEV_ROUTINE) {
        char apptStr[60];
        apptToString(&p->appt, apptStr, sizeof(apptStr));
        printf("  [i] Slot '%s' is now available for booking.\n", apptStr);
    }
}

/* ------------------------------------------------------------------ */
/*  Core operations                                                     */
/* ------------------------------------------------------------------ */

void addPatient() {
    if (heap.size >= MAX_PATIENTS) {
        printf("\n  [!] Queue is full!\n");
        return;
    }
    Patient p;
    memset(&p, 0, sizeof(p));
    p.id = nextId++;

   printf("\n--- Add New Patient (ID: %d) ---\n", p.id);

   do {
        readString(p.ownerName, NAME_LEN, "  Owner Name : ");
        if (strlen(p.ownerName) == 0)
            printf("  [!] Please enter owner name.\n");
    } while (strlen(p.ownerName) == 0);

   if (linearSearchByName(p.ownerName) >= 0)
        printf("  [!] Warning: A patient with a similar owner name is already in the queue.\n");

   do {
        readString(p.petName, NAME_LEN, "  Pet Name   : ");
        if (strlen(p.petName) == 0)
            printf("  [!] Please enter a pet name.\n");
    } while (strlen(p.petName) == 0);

   do {
        readString(p.petType, NAME_LEN, "  Pet Type (e.g. Dog, Cat, Parrot, Snake): ");
        if (strlen(p.petType) == 0)
            printf("  [!] Please enter a pet type.\n");
    } while (strlen(p.petType) == 0);

   p.severity = getSeverity();

  printf("\n  --- Schedule Appointment ---\n");
    p.appt = getAppointmentInfo();

   /* Check if this date+time slot is already taken (Routine/Moderate only) */
  {
        int takenById = isSlotTaken(&p.appt, p.severity);
        if (takenById >= 0) {
            char apptStr[60];
            apptToString(&p.appt, apptStr, sizeof(apptStr));
            printf("\n  [!] This slot is unavailable: %s\n", apptStr);
            printf("      Already booked by Patient ID %d.\n", takenById);
            printf("      This slot will be free once that patient is served.\n");
            printf("      Please choose a different date or time.\n");
            nextId--;  /* roll back the ID counter */
            return;
        }
    }

  p.enrolledAt    = time(NULL);
    p.waitMinutes   = 0;
    p.priorityScore = computePriority(p.severity, p.waitMinutes, p.appt.hhmm);

   heapEnqueue(p);
    stackPush(p);
    bstRoot = bstInsert(bstRoot, p);

  char apptStr[60];
    apptToString(&p.appt, apptStr, sizeof(apptStr));
    printf("\n  [+] Patient '%s' (Pet: %s) added.\n", p.ownerName, p.petName);
    printf("      Appointment   : %s\n", apptStr);
    printf("      Priority Score: %d | Severity: %s\n",
           p.priorityScore, severityLabel(p.severity));
}

void serveNext() {
    if (heap.size == 0) {
        printf("\n  [!] No patients in queue.\n");
        return;
    }
    Patient served      = heapDequeue();
    served.waitMinutes  = getElapsedMinutes(served.enrolledAt);
    listPrepend(&history, served);
    releaseSlot(&served);

  char apptStr[60];
  apptToString(&served.appt, apptStr, sizeof(apptStr));

   printf("\n");
    printDivider('*', 55);
    printf("  NOW SERVING\n");
    printDivider('*', 55);
    printf("  Patient ID    : %d\n",      served.id);
    printf("  Owner         : %s\n",      served.ownerName);
    printf("  Pet           : %s (%s)\n", served.petName, served.petType);
    printf("  Severity      : %s\n",      severityLabel(served.severity));
    printf("  Appointment   : %s\n",      apptStr);
    printf("  Wait Time     : %d min\n",  served.waitMinutes);
    printf("  Priority Score: %d\n",      served.priorityScore);
    printDivider('*', 55);
    printf("  Remaining in queue: %d\n", heap.size);
}

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
        BSTNode *found = bstSearch(bstRoot, id);
        if (found) {
            Patient *p = &found->patient;
            char apptStr[60];
            apptToString(&p->appt, apptStr, sizeof(apptStr));
            printf("\n  [BST Binary Search] Found in records:\n");
            printf("  ID: %d | Owner: %s | Pet: %s (%s)\n",
                   p->id, p->ownerName, p->petName, p->petType);
            printf("  Severity: %s | Appointment: %s | Score: %d\n",
                   severityLabel(p->severity), apptStr, p->priorityScore);
            printf("  Status: %s\n",
                   linearSearchById(id) >= 0 ? "In Queue" : "Already served or removed");
        } else {
            int idx = linearSearchById(id);
            if (idx >= 0) {
                Patient *p = &heap.data[idx];
                char apptStr[60];
                apptToString(&p->appt, apptStr, sizeof(apptStr));
                printf("\n  [Linear Search] Found in queue:\n");
                printf("  ID: %d | Owner: %s | Pet: %s | Appt: %s | Score: %d\n",
                       p->id, p->ownerName, p->petName, apptStr, p->priorityScore);
            } else {
                printf("\n  [!] Patient with ID %d not found.\n", id);
            }
        }
    } else if (choice == 2) {
        char name[NAME_LEN];
        readString(name, NAME_LEN, "  Enter Owner Name (partial ok): ");
        int idx = linearSearchByName(name);
        if (idx >= 0) {
            Patient *p = &heap.data[idx];
            char apptStr[60];
            apptToString(&p->appt, apptStr, sizeof(apptStr));
            printf("\n  [Linear Search] Found in queue:\n");
            printf("  ID: %d | Owner: %s | Pet: %s (%s)\n",
                   p->id, p->ownerName, p->petName, p->petType);
            printf("  Severity: %s | Appointment: %s | Score: %d\n",
                   severityLabel(p->severity), apptStr, p->priorityScore);
        } else {
            printf("\n  [!] No patient with owner name containing '%s' found in queue.\n", name);
            printf("      (They may have already been served — check History)\n");
        }
    } else {
        printf("  [!] Invalid choice.\n");
    }
}

void viewBSTRecords() {
    printf("\n--- All Patient Records (BST In-Order by ID) ---\n");
    if (!bstRoot) {
        printf("  [No records yet]\n");
        return;
    }
    printf("  %-5s %-20s %-15s %-10s %-10s %-35s %-6s %-5s\n",
           "ID","Owner","Pet Name","Type","Severity","Appointment","Wait","Score");
    printDivider('-', 110);
    int count = 0;
    bstInOrder(bstRoot, &count);
    printf("\n  Total records: %d\n", count);
}

void undoLastAdd() {
    Patient p;
    if (!stackPop(&p)) {
        printf("\n  [!] Nothing to undo.\n");
        return;
    }
    int found = -1;
    for (int i = 0; i < heap.size; i++) {
        if (heap.data[i].id == p.id) { found = i; break; }
    }
    if (found >= 0) {
        heap.data[found] = heap.data[--heap.size];
        if (found > 0 &&
            heap.data[found].priorityScore < heap.data[(found - 1) / 2].priorityScore)
            heapifyUp(found);
        else
            heapifyDown(found);
        bstRoot = bstDelete(bstRoot, p.id);
        releaseSlot(&p);
        printf("\n  [Undo] Removed patient: %s (Pet: %s, ID: %d)\n",
               p.ownerName, p.petName, p.id);
    } else {
        printf("\n  [!] Patient already served — cannot undo.\n");
    }
}

/* ------------------------------------------------------------------ */
/*  Cancel Appointment                                                  */
/* ------------------------------------------------------------------ */

void cancelAppointment() {
    printf("\n--- Cancel Appointment ---\n");

  if (heap.size == 0) {
        printf("  [!] No patients in queue to cancel.\n");
        return;
    }

   int id;
    printf("  Enter Patient ID to cancel: ");
    if (scanf("%d", &id) != 1) { clearInput(); return; }
    clearInput();

  /* Linear search in heap */
    int found = -1;
    for (int i = 0; i < heap.size; i++) {
        if (heap.data[i].id == id) { found = i; break; }
    }

  if (found < 0) {
        printf("  [!] Patient ID %d not found in queue.\n", id);
        printf("      (Already served, cancelled, or does not exist)\n");
        return;
    }

   /* Show patient details */
    Patient *p = &heap.data[found];
    char apptStr[60];
    apptToString(&p->appt, apptStr, sizeof(apptStr));

  printf("\n  Patient found:\n");
    printf("  ID          : %d\n",      p->id);
    printf("  Owner       : %s\n",      p->ownerName);
    printf("  Pet         : %s (%s)\n", p->petName, p->petType);
    printf("  Severity    : %s\n",      severityLabel(p->severity));
    printf("  Appointment : %s\n",      apptStr);

   /* Confirm */
    char confirm;
    printf("\n  Are you sure you want to cancel? [Y/N]: ");
    scanf(" %c", &confirm);
    clearInput();

   if (confirm != 'Y' && confirm != 'y') {
        printf("  [!] Cancellation aborted.\n");
        return;
    }

  /* Save copy then remove from heap */
    Patient removed = heap.data[found];
    heap.data[found] = heap.data[--heap.size];
    if (heap.size > 0) {
        if (found > 0 &&
            heap.data[found].priorityScore < heap.data[(found-1)/2].priorityScore)
            heapifyUp(found);
        else
            heapifyDown(found);
    }

  /* Remove from BST */
    bstRoot = bstDelete(bstRoot, removed.id);

  /* Release Sunday slot if this patient held it */
    releaseSlot(&removed);

   /* Add to cancelled linked list */
    listPrepend(&cancelled, removed);

   printf("\n  [✓] Appointment #%d for '%s' (Pet: %s) has been cancelled.\n",
           removed.id, removed.ownerName, removed.petName);
    printf("      Remaining in queue: %d\n", heap.size);
}

/* ------------------------------------------------------------------ */
/*  View Cancelled Appointments                                         */
/* ------------------------------------------------------------------ */

void viewCancelled() {
    printf("\n--- Cancelled Appointments ---\n");
    if (!cancelled.head) {
        printf("  [No cancelled appointments]\n");
        return;
    }
    printf("  %-4s %-5s %-20s %-15s %-10s %-10s %-35s\n",
           "No.","ID","Owner","Pet","Type","Severity","Appointment");
    printDivider('-', 105);
    HistoryNode *cur = cancelled.head;
    int num = 1;
    while (cur) {
        Patient *p = &cur->patient;
        char apptStr[60];
        apptToString(&p->appt, apptStr, sizeof(apptStr));
        printf("  %-4d %-5d %-20s %-15s %-10s %-10s %-35s\n",
               num++, p->id, p->ownerName, p->petName,
               p->petType, severityLabel(p->severity), apptStr);
        cur = cur->next;
    }
    printf("\n  Total cancelled: %d\n", cancelled.count);
}

/* ------------------------------------------------------------------ */
/*  main                                                                */
/* ------------------------------------------------------------------ */

int main() {
    printHeader();
    printf("   DSA Used: Min-Heap | Linked List | Stack | BST\n");
    printf("   Algorithms: Bubble Sort | Linear Search | BST Search\n");
    printDivider('=', 54);

  int choice;
    while (1) {
        printMainMenu();
        if (scanf("%d", &choice) != 1) { clearInput(); continue; }
        clearInput();
        switch (choice) {
        case 1: addPatient();      break;
        case 2: serveNext();       break;
        case 3: {
            if (heap.size == 0) { printf("\n  [Queue is empty]\n"); break; }
            int sortMode = SORT_PRIORITY;
            printSortMenu();
            if (scanf("%d", &sortMode) != 1 || sortMode < 0 || sortMode > 4)
                sortMode = SORT_PRIORITY;
            clearInput();
            printf("\n--- Current Queue ---");
            displayQueue(sortMode);
            break;
        }
        case 4: searchPatient();   break;
        case 5: viewBSTRecords();  break;
        case 6:
            printf("\n--- Patient History (Most Recently Served First) ---");
            listPrint();
            break;
        case 7: undoLastAdd();     break;
        case 8: cancelAppointment(); break;
        case 9: viewCancelled();   break;
        case 0:
            printf("\n  Goodbye! Stay healthy, pets!\n\n");
            listFreeAll(&history);
            listFreeAll(&cancelled);
            bstFree(bstRoot);
            return 0;
        default:
            printf("\n  [!] Invalid choice. Enter 0-9.\n");
        }
    }
}
