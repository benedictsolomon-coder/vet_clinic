Overview and Purpose
The Veterinary Clinic Appointment Management System is a console-based application written in C language that helps clinic staff manage patient queues and appointments efficiently. When a pet owner visits or contacts the clinic, a staff member registers them by entering the owner's name, pet name, pet type, severity level (Critical, Urgent, Moderate, or Routine), and a preferred appointment date and time. The system then automatically computes a priority score for each patient and places them in a queue, ensuring that the most critical cases are always attended first while still being fair to patients who have been waiting a long time.

The priority score is calculated using the formula priorityScore = severityScore - waitScore + apptScore. The severity score assigns a base value of 0 for Critical, 10 for Urgent, 20 for Moderate, and 30 for Routine. The wait score, which is subtracted, rewards patients who have been waiting longer by lowering their score, making them rise in the queue over time. A lower total score means higher priority, so the patient at the top of the queue is always the one who needs to be seen most urgently. The system supports up to 100 active patients in the queue at a time and keeps a permanent record of every patient ever registered, including those already served or appointments that had been cancelled.

Data Structures and Algorithms
The system uses four core data structures together with bubble sort and linear search. The min-heap serves as the live patient queue, always keeping the most urgent patient at the top so the system can instantly find who to serve next, it also supports adding patients, serving them, cancelling appointments, and undoing additions. Two singly linked lists maintain the served patient history and the cancelled appointments section, where each new entry is simply attached to the front of the list so the most recent record always appears first. An array-based stack enables undo functionality by saving a copy of each patient as they are added, so if the last addition needs to be reversed, it can be removed quickly and cleanly. A binary search tree (BST) focused on patient ID keeps a permanent record of every registered patient, organized in a way that makes looking up, adding, and removing records fast without scanning the entire list. For display purposes, bubble sort is applied to a temporary copy of the queue, never the real one to offer five sort modes (priority, severity, wait time, appointment time, and name), while linear search handles name-based lookups, appointment slot conflict checks, and ID searches when a quick BST lookup is not enough.

Features
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

Compilationa and Running
This is how you compile and run it in terminal or your command prompt (CMD) for Windows, macOS, and Linux.

"gcc -o vet_clinic vet_clinic.c" type this on your terminal to compile it and for you to be able to run the program.

"./vet_clinic" use this command to run the program in macOS, and Linux.

"vet_clinic.exe" use this command to run the program in Windows.
