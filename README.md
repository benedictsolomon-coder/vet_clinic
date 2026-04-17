# vet_clinic

/*
 * ============================================================
 *   VET CLINIC APPOINTMENT MANAGEMENT SYSTEM
 *   CC104: Data Structures and Algorithms — Final Project
 * ============================================================
 *
 *  DATA STRUCTURES USED:
 *  1. Min-Heap (Priority Queue) — manages live appointment queue
 *  2. Linked List               — stores served patient history
 *  3. Stack                     — undo last added patient
 *  4. Binary Search Tree (BST)  — organizes all patients by ID
 *                                 for fast lookup and in-order display
 *
 *  ALGORITHMS USED:
 *  1. Heap Sort (Selection via Heapify) — dequeue always returns
 *     the highest-priority patient in O(log n)
 *  2. Bubble Sort — sort queue display by different criteria
 *     (severity, wait time, appointment time, name)
 *  3. Linear Search — find patient in queue by ID or owner name
 *     Binary Search  — find patient in BST by ID (O(log n))
 */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <ctype.h>

// ──────────────────────────────────────────────
// CONSTANTS
// ──────────────────────────────────────────────
#define MAX_PATIENTS  100
#define NAME_LEN      50
#define STACK_SIZE    100

#define SEV_CRITICAL  1
#define SEV_URGENT    2
#define SEV_MODERATE  3
#define SEV_ROUTINE   4

// Sort modes for display
#define SORT_PRIORITY  0
#define SORT_SEVERITY  1
#define SORT_WAIT      2
#define SORT_APPT      3
#define SORT_NAME      4

// ──────────────────────────────────────────────
// PATIENT STRUCT (shared across all structures)
// ──────────────────────────────────────────────
typedef struct {
    int  id;
    char ownerName[NAME_LEN];
    char petName[NAME_LEN];
    char petType[NAME_LEN];
    int  isExotic;
    int  severity;
    int  appointmentTime;   // HHMM format
    int  waitMinutes;
    int  priorityScore;     // lower = higher priority
} Patient;

// ──────────────────────────────────────────────
// DATA STRUCTURE 1: MIN-HEAP (Priority Queue)
// Used for: live appointment queue
// enqueue = O(log n), dequeue = O(log n)
// ──────────────────────────────────────────────
typedef struct {
    Patient data[MAX_PATIENTS];
    int     size;
} MinHeap;

// ──────────────────────────────────────────────
// DATA STRUCTURE 2: LINKED LIST (Patient History)
// Used for: storing served patients in order served
// insert = O(1) at head, traversal = O(n)
// ──────────────────────────────────────────────
typedef struct HistoryNode {
    Patient             patient;
    struct HistoryNode *next;
} HistoryNode;

typedef struct {
    HistoryNode *head;
    int          count;
} LinkedList;

// ──────────────────────────────────────────────
// DATA STRUCTURE 3: STACK (Undo Last Add)
// Used for: undoing the last patient added to the queue
// push = O(1), pop = O(1)
// ──────────────────────────────────────────────
typedef struct {
    Patient data[STACK_SIZE];
    int     top;  // -1 when empty
} Stack;

// ──────────────────────────────────────────────
// DATA STRUCTURE 4: BINARY SEARCH TREE (BST)
// Used for: organized patient record lookup by ID
// insert = O(log n) avg, search = O(log n) avg
// in-order traversal gives sorted list by ID
// ──────────────────────────────────────────────
typedef struct BSTNode {
    Patient         patient;
    struct BSTNode *left;
    struct BSTNode *right;
} BSTNode;

// ──────────────────────────────────────────────
// GLOBAL STATE
// ──────────────────────────────────────────────
MinHeap    heap    = { .size = 0 };
LinkedList history = { .head = NULL, .count = 0 };
Stack      undoStk = { .top = -1 };
BSTNode   *bstRoot = NULL;
int        nextId  = 1;

// ──────────────────────────────────────────────
// PRIORITY SCORE CALCULATION
// Lower score = served sooner
//   Severity   : 0–30 (Critical→0, Routine→30)
//   Pet type   : 0–5  (Exotic→0, Common→5)
//   Wait time  : 0–20 (longer wait → lower penalty)
//   Appt time  : 0–15 (earlier appt → lower score)
// ──────────────────────────────────────────────
int computePriority(int severity, int isExotic, int waitMinutes, int appointmentTime) {
    int sevScore;
    switch (severity) {
        case SEV_CRITICAL: sevScore =  0; break;
        case SEV_URGENT:   sevScore = 10; break;
        case SEV_MODERATE: sevScore = 20; break;
        default:           sevScore = 30; break;
    }

    int petScore  = isExotic ? 0 : 5;

    // Longer wait = lower score (bonus capped at 20)
    int waitScore = 20 - (waitMinutes / 3);
    if (waitScore < 0)  waitScore = 0;
    if (waitScore > 20) waitScore = 20;

    // Earlier appointment = lower score (0–15)
    int apptMins  = (appointmentTime / 100) * 60 + (appointmentTime % 100);
    int apptScore = apptMins / 64;
    if (apptScore > 15) apptScore = 15;

    return sevScore + petScore + waitScore + apptScore;
}

// ──────────────────────────────────────────────
// HELPERS
// ──────────────────────────────────────────────
const char *severityLabel(int s) {
    switch (s) {
        case SEV_CRITICAL: return "CRITICAL";
        case SEV_URGENT:   return "URGENT";
        case SEV_MODERATE: return "MODERATE";
        default:           return "ROUTINE";
    }
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

// ──────────────────────────────────────────────
// MIN-HEAP OPERATIONS (Data Structure 1)
// ──────────────────────────────────────────────
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

// Insert into heap — O(log n)
void heapEnqueue(Patient p) {
    if (heap.size >= MAX_PATIENTS) {
        printf("  [!] Queue is full.\n");
        return;
    }
    p.priorityScore = computePriority(p.severity, p.isExotic, p.waitMinutes, p.appointmentTime);
    heap.data[heap.size++] = p;
    heapifyUp(heap.size - 1);
}

// Remove highest-priority patient — O(log n)
Patient heapDequeue() {
    Patient top   = heap.data[0];
    heap.data[0]  = heap.data[--heap.size];
    heapifyDown(0);
    return top;
}

// ──────────────────────────────────────────────
// LINKED LIST OPERATIONS (Data Structure 2)
// ──────────────────────────────────────────────

// Prepend served patient to history — O(1)
void listPrepend(Patient p) {
    HistoryNode *node = (HistoryNode *)malloc(sizeof(HistoryNode));
    if (!node) { printf("  [!] Memory error.\n"); return; }
    node->patient = p;
    node->next    = history.head;
    history.head  = node;
    history.count++;
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

// Free all history nodes
void listFree() {
    HistoryNode *cur = history.head;
    while (cur) {
        HistoryNode *tmp = cur;
        cur = cur->next;
        free(tmp);
    }
    history.head  = NULL;
    history.count = 0;
}

// ──────────────────────────────────────────────
// STACK OPERATIONS (Data Structure 3)
// ──────────────────────────────────────────────

// Push patient onto undo stack — O(1)
void stackPush(Patient p) {
    if (undoStk.top >= STACK_SIZE - 1) {
        // Shift oldest out to make room
        for (int i = 0; i < STACK_SIZE - 1; i++)
            undoStk.data[i] = undoStk.data[i + 1];
        undoStk.data[undoStk.top] = p;
    } else {
        undoStk.data[++undoStk.top] = p;
    }
}

// Pop from undo stack — O(1)
int stackPop(Patient *out) {
    if (undoStk.top < 0) return 0;
    *out = undoStk.data[undoStk.top--];
    return 1;
}

int stackIsEmpty() { return undoStk.top < 0; }

// ──────────────────────────────────────────────
// BST OPERATIONS (Data Structure 4)
// Binary Search Tree keyed on patient ID
// ──────────────────────────────────────────────

// Allocate a new BST node
BSTNode *bstNewNode(Patient p) {
    BSTNode *n = (BSTNode *)malloc(sizeof(BSTNode));
    if (!n) { printf("  [!] Memory error.\n"); return NULL; }
    n->patient = p;
    n->left = n->right = NULL;
    return n;
}

// Recursive insert — O(log n) average
BSTNode *bstInsert(BSTNode *root, Patient p) {
    if (!root) return bstNewNode(p);
    if (p.id < root->patient.id)
        root->left  = bstInsert(root->left,  p);
    else if (p.id > root->patient.id)
        root->right = bstInsert(root->right, p);
    // Duplicate IDs: update in place
    else
        root->patient = p;
    return root;
}

// Binary search in BST by ID — O(log n) average
BSTNode *bstSearch(BSTNode *root, int id) {
    if (!root || root->patient.id == id) return root;
    if (id < root->patient.id) return bstSearch(root->left,  id);
    else                       return bstSearch(root->right, id);
}

// In-order traversal prints patients sorted by ID
void bstInOrder(BSTNode *root, int *count) {
    if (!root) return;
    bstInOrder(root->left, count);
    Patient *p = &root->patient;
    (*count)++;
    printf("  %-5d %-20s %-15s %-10s %-10s %04d      %-6d %-5d\n",
           p->id, p->ownerName, p->petName, p->petType,
           severityLabel(p->severity), p->appointmentTime,
           p->waitMinutes, p->priorityScore);
    bstInOrder(root->right, count);
}

// Find and remove a node from BST by ID (used for undo)
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
        // Node found
        if (!root->left) {
            BSTNode *tmp = root->right; free(root); return tmp;
        } else if (!root->right) {
            BSTNode *tmp = root->left;  free(root); return tmp;
        }
        // Two children: replace with in-order successor
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

// ──────────────────────────────────────────────
// ALGORITHM 1: BUBBLE SORT
// Sorts a copy of the heap array by chosen criterion
// Time complexity: O(n²) — fine for n ≤ 100
// ──────────────────────────────────────────────
int comparePatients(Patient *a, Patient *b, int mode) {
    switch (mode) {
        case SORT_SEVERITY: return a->severity   - b->severity;
        case SORT_WAIT:     return b->waitMinutes - a->waitMinutes; // descending
        case SORT_APPT:     return a->appointmentTime - b->appointmentTime;
        case SORT_NAME:     return strcmp(a->ownerName, b->ownerName);
        default:            return a->priorityScore - b->priorityScore;
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
        if (!swapped) break; // already sorted early exit
    }
}

// ──────────────────────────────────────────────
// ALGORITHM 2: LINEAR SEARCH
// Searches the heap array for a patient by ID or name
// Time complexity: O(n)
// ──────────────────────────────────────────────
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

// ──────────────────────────────────────────────
// ALGORITHM 3: BINARY SEARCH (in BST)
// bstSearch() above performs O(log n) search on BST
// Here we also do binary search on a sorted array
// for the history log (sorted by arrival order = linked list)
// and demonstrate it explicitly for the records view.
// ──────────────────────────────────────────────
// (bstSearch is the primary binary search; see BST section above)

// ──────────────────────────────────────────────
// DISPLAY QUEUE
// ──────────────────────────────────────────────
void displayQueue(int sortMode) {
    if (heap.size == 0) {
        printf("\n  [Queue is empty]\n");
        return;
    }

    // Work on a sorted copy — never disturb the actual heap
    Patient copy[MAX_PATIENTS];
    memcpy(copy, heap.data, heap.size * sizeof(Patient));
    bubbleSort(copy, heap.size, sortMode); // Algorithm: Bubble Sort

    const char *sortLabel[] = {
        "Priority Score","Severity","Wait Time","Appointment Time","Owner Name"
    };
    printf("\n  Sorted by: %s\n", sortLabel[sortMode]);
    printf("  %-4s %-5s %-20s %-15s %-10s %-10s %-9s %-6s %-6s\n",
           "Rank","ID","Owner","Pet","Type","Severity","ApptTime","Wait","Score");
    printDivider('-', 92);

    for (int i = 0; i < heap.size; i++) {
        Patient *p = &copy[i];
        printf("  %-4d %-5d %-20s %-15s %-10s %-10s %04d      %-6d %-6d\n",
               i + 1, p->id, p->ownerName, p->petName, p->petType,
               severityLabel(p->severity), p->appointmentTime,
               p->waitMinutes, p->priorityScore);
    }
    printf("\n  Patients in queue: %d\n", heap.size);
}

// ──────────────────────────────────────────────
// INPUT HELPERS
// ──────────────────────────────────────────────
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

int getAppointmentTime() {
    int t;
    printf("  Appointment Time (HHMM, e.g. 0900): ");
    while (scanf("%d", &t) != 1 || t < 0 || t > 2359 || (t % 100) >= 60) {
        printf("  Invalid time. Try again: ");
        clearInput();
    }
    clearInput();
    return t;
}

// ──────────────────────────────────────────────
// MENUS
// ──────────────────────────────────────────────
void printHeader() {
    printf("\n");
    printDivider('=', 54);
    printf("   VETCARE CLINIC — APPOINTMENT MANAGEMENT SYSTEM\n");
    printDivider('=', 54);
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
    printf("  |  8. Exit                             |\n");
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

// ──────────────────────────────────────────────
// FEATURE: ADD PATIENT
// ──────────────────────────────────────────────
void addPatient() {
    if (heap.size >= MAX_PATIENTS) {
        printf("\n  [!] Queue is full!\n");
        return;
    }

    Patient p;
    p.id = nextId++;

    printf("\n--- Add New Patient (ID: %d) ---\n", p.id);

    printf("  Owner Name : ");
    fgets(p.ownerName, NAME_LEN, stdin);
    p.ownerName[strcspn(p.ownerName, "\n")] = '\0';
    if (strlen(p.ownerName) == 0) {
        strncpy(p.ownerName, "Unknown", NAME_LEN);
    }

    printf("  Pet Name   : ");
    fgets(p.petName, NAME_LEN, stdin);
    p.petName[strcspn(p.petName, "\n")] = '\0';
    if (strlen(p.petName) == 0) {
        strncpy(p.petName, "Unknown", NAME_LEN);
    }

    printf("  Pet Type (e.g. Dog, Cat, Parrot, Snake): ");
    fgets(p.petType, NAME_LEN, stdin);
    p.petType[strcspn(p.petType, "\n")] = '\0';
    if (strlen(p.petType) == 0) {
        strncpy(p.petType, "Unknown", NAME_LEN);
    }

    printf("  Is exotic pet? (1=Yes, 0=No): ");
    while (scanf("%d", &p.isExotic) != 1 || (p.isExotic != 0 && p.isExotic != 1)) {
        printf("  Enter 0 or 1: ");
        clearInput();
    }
    clearInput();

    p.severity        = getSeverity();
    p.appointmentTime = getAppointmentTime();

    printf("  Minutes already waited: ");
    while (scanf("%d", &p.waitMinutes) != 1 || p.waitMinutes < 0) {
        printf("  Enter a non-negative number: ");
        clearInput();
    }
    clearInput();

    p.priorityScore = computePriority(p.severity, p.isExotic, p.waitMinutes, p.appointmentTime);

    // Add to Min-Heap (Data Structure 1)
    heapEnqueue(p);

    // Push to Undo Stack (Data Structure 3)
    stackPush(p);

    // Insert into BST (Data Structure 4)
    bstRoot = bstInsert(bstRoot, p);

    printf("\n  [+] Patient '%s' (Pet: %s) added.\n", p.ownerName, p.petName);
    printf("      Priority Score: %d | Severity: %s\n",
           p.priorityScore, severityLabel(p.severity));
}

// ──────────────────────────────────────────────
// FEATURE: SERVE NEXT PATIENT
// ──────────────────────────────────────────────
void serveNext() {
    if (heap.size == 0) {
        printf("\n  [!] No patients in queue.\n");
        return;
    }

    // Dequeue from heap (Algorithm: Heap-based priority extraction)
    Patient served = heapDequeue();

    // Add to linked list history (Data Structure 2)
    listPrepend(served);

    printf("\n");
    printDivider('*', 50);
    printf("  NOW SERVING\n");
    printDivider('*', 50);
    printf("  Patient ID    : %d\n",   served.id);
    printf("  Owner         : %s\n",   served.ownerName);
    printf("  Pet           : %s (%s%s)\n",
           served.petName, served.petType,
           served.isExotic ? ", Exotic" : "");
    printf("  Severity      : %s\n",   severityLabel(served.severity));
    printf("  Appt Time     : %04d\n", served.appointmentTime);
    printf("  Wait Time     : %d min\n", served.waitMinutes);
    printf("  Priority Score: %d\n",   served.priorityScore);
    printDivider('*', 50);
    printf("  Remaining in queue: %d\n", heap.size);
}

// ──────────────────────────────────────────────
// FEATURE: SEARCH PATIENT
// Uses: Linear Search (Algorithm 2) in heap
//       Binary Search via BST (Algorithm 3) by ID
// ──────────────────────────────────────────────
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

        // Binary Search in BST (Algorithm 3) — O(log n)
        BSTNode *found = bstSearch(bstRoot, id);
        if (found) {
            Patient *p = &found->patient;
            printf("\n  [BST Binary Search] Found in records:\n");
            printf("  ID: %d | Owner: %s | Pet: %s (%s) | Severity: %s | Score: %d\n",
                   p->id, p->ownerName, p->petName, p->petType,
                   severityLabel(p->severity), p->priorityScore);

            // Also check if still in queue via Linear Search
            int idx = linearSearchById(id);
            if (idx >= 0)
                printf("  Status: Still in queue (position not guaranteed — heap order)\n");
            else
                printf("  Status: Already served or removed\n");
        } else {
            // Linear search as fallback in queue
            int idx = linearSearchById(id);
            if (idx >= 0) {
                Patient *p = &heap.data[idx];
                printf("\n  [Linear Search] Found in queue:\n");
                printf("  ID: %d | Owner: %s | Pet: %s | Score: %d\n",
                       p->id, p->ownerName, p->petName, p->priorityScore);
            } else {
                printf("\n  [!] Patient with ID %d not found.\n", id);
            }
        }

    } else if (choice == 2) {
        char name[NAME_LEN];
        printf("  Enter Owner Name (partial ok): ");
        fgets(name, NAME_LEN, stdin);
        name[strcspn(name, "\n")] = '\0';

        // Linear Search by name (Algorithm 2) — O(n)
        int idx = linearSearchByName(name);
        if (idx >= 0) {
            Patient *p = &heap.data[idx];
            printf("\n  [Linear Search] Found in queue:\n");
            printf("  ID: %d | Owner: %s | Pet: %s (%s) | Severity: %s | Score: %d\n",
                   p->id, p->ownerName, p->petName, p->petType,
                   severityLabel(p->severity), p->priorityScore);
        } else {
            printf("\n  [!] No patient with owner name containing '%s' found in queue.\n", name);
            printf("      (They may have already been served — check History)\n");
        }
    } else {
        printf("  Invalid choice.\n");
    }
}

// ──────────────────────────────────────────────
// FEATURE: VIEW BST RECORDS (in-order = sorted by ID)
// ──────────────────────────────────────────────
void viewBSTRecords() {
    printf("\n--- All Patient Records (BST In-Order by ID) ---\n");
    if (!bstRoot) {
        printf("  [No records yet]\n");
        return;
    }
    printf("  %-5s %-20s %-15s %-10s %-10s %-9s %-6s %-5s\n",
           "ID","Owner","Pet","Type","Severity","ApptTime","Wait","Score");
    printDivider('-', 85);
    int count = 0;
    bstInOrder(bstRoot, &count);
    printf("\n  Total records: %d\n", count);
}

// ──────────────────────────────────────────────
// FEATURE: UNDO LAST ADD
// Pops from undo stack, removes from heap and BST
// ──────────────────────────────────────────────
void undoLastAdd() {
    Patient p;
    if (!stackPop(&p)) {
        printf("\n  [!] Nothing to undo.\n");
        return;
    }

    // Remove from heap by finding and extracting
    int found = -1;
    for (int i = 0; i < heap.size; i++) {
        if (heap.data[i].id == p.id) { found = i; break; }
    }

    if (found >= 0) {
        // Replace with last element and re-heapify
        heap.data[found] = heap.data[--heap.size];
        // Restore heap property
        heapifyDown(found);
        heapifyUp(found);

        // Remove from BST
        bstRoot = bstDelete(bstRoot, p.id);

        printf("\n  [Undo] Removed patient: %s (Pet: %s, ID: %d)\n",
               p.ownerName, p.petName, p.id);
        // Reclaim the ID
        nextId--;
    } else {
        printf("\n  [!] Patient already served — cannot undo.\n");
    }
}

// ──────────────────────────────────────────────
// MAIN
// ──────────────────────────────────────────────
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

        case 1:
            addPatient();
            break;

        case 2:
            serveNext();
            break;

        case 3: {
            if (heap.size == 0) {
                printf("\n  [Queue is empty]\n");
                break;
            }
            int sortMode = SORT_PRIORITY;
            printSortMenu();
            if (scanf("%d", &sortMode) != 1 || sortMode < 0 || sortMode > 4)
                sortMode = SORT_PRIORITY;
            clearInput();
            printf("\n--- Current Queue ---");
            displayQueue(sortMode);
            break;
        }

        case 4:
            searchPatient();
            break;

        case 5:
            viewBSTRecords();
            break;

        case 6:
            printf("\n--- Patient History (Most Recently Served First) ---");
            listPrint();
            break;

        case 7:
            undoLastAdd();
            break;

        case 8:
            printf("\n  Goodbye! Stay healthy, pets!\n\n");
            listFree();
            bstFree(bstRoot);
            return 0;

        default:
            printf("\n  Invalid choice. Enter 1-8.\n");
        }
    }
}
