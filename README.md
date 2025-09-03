# Student-schedule-planner
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Exam Schedule Planner</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <!-- Firebase SDKs -->
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, doc, onSnapshot, setDoc, getDoc, updateDoc } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";
        import { setLogLevel } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        // Firebase-related variables will be defined by the environment
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
        const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {};
        const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;

        // Initialize Firebase
        const app = initializeApp(firebaseConfig);
        const auth = getAuth(app);
        const db = getFirestore(app);
        // Set log level for debugging
        setLogLevel('debug');

        let userId = null;
        let userSchedulesDocRef = null;

        // UI Elements
        const subjectInput = document.getElementById('subjectInput');
        const examDateInput = document.getElementById('examDateInput');
        const scheduleContainer = document.getElementById('scheduleContainer');
        const userIdDisplay = document.getElementById('userIdDisplay');
        const messageBox = document.getElementById('messageBox');
        const messageText = document.getElementById('messageText');
        const messageCloseBtn = document.getElementById('messageCloseBtn');

        // Function to show a custom message box instead of alert()
        const showMessage = (text) => {
            messageText.textContent = text;
            messageBox.classList.remove('hidden');
        };

        messageCloseBtn.addEventListener('click', () => {
            messageBox.classList.add('hidden');
        });

        // Add a subject to the schedule
        const addSubject = async () => {
            const subjectName = subjectInput.value.trim();
            const examDate = examDateInput.value;

            if (subjectName && examDate) {
                try {
                    // Get current data from Firestore
                    const docSnap = await getDoc(userSchedulesDocRef);
                    const currentData = docSnap.exists() ? docSnap.data() : { subjects: [] };

                    const newSubject = {
                        name: subjectName,
                        examDate: examDate,
                        tasks: []
                    };
                    currentData.subjects.push(newSubject);

                    // Save the updated data back to Firestore
                    await setDoc(userSchedulesDocRef, currentData);
                    subjectInput.value = '';
                    examDateInput.value = '';
                } catch (e) {
                    console.error("Error adding subject: ", e);
                    showMessage("Error adding subject. Please try again.");
                }
            } else {
                showMessage("Please enter both a subject name and an exam date.");
            }
        };

        // Add a task to a specific subject
        const addTask = async (subjectIndex, taskDescription) => {
            if (taskDescription) {
                try {
                    const docSnap = await getDoc(userSchedulesDocRef);
                    if (docSnap.exists()) {
                        const currentData = docSnap.data();
                        const subject = currentData.subjects[subjectIndex];
                        if (subject) {
                            const newDate = new Date().toISOString().split('T')[0];
                            subject.tasks.push({
                                description: taskDescription,
                                date: newDate
                            });
                            await updateDoc(userSchedulesDocRef, {
                                subjects: currentData.subjects
                            });
                        }
                    }
                } catch (e) {
                    console.error("Error adding task: ", e);
                    showMessage("Error adding task. Please try again.");
                }
            }
        };

        // Render the schedule from Firestore data
        const renderSchedule = (data) => {
            scheduleContainer.innerHTML = '';
            const subjects = data.subjects || [];

            if (subjects.length === 0) {
                scheduleContainer.innerHTML = '<p class="text-center text-gray-500 mt-8">No subjects added yet. Start planning!</p>';
                return;
            }

            subjects.sort((a, b) => new Date(a.examDate) - new Date(b.examDate));

            subjects.forEach((subject, subjectIndex) => {
                const subjectCard = document.createElement('div');
                subjectCard.className = 'bg-white p-6 rounded-lg shadow-md mb-6';
                subjectCard.innerHTML = `
                    <div class="flex flex-col sm:flex-row items-start sm:items-center justify-between mb-4">
                        <h2 class="text-2xl font-semibold text-gray-800">${subject.name}</h2>
                        <span class="text-sm font-medium text-purple-600 bg-purple-100 px-3 py-1 rounded-full mt-2 sm:mt-0">Exam Date: ${subject.examDate}</span>
                    </div>
                    <div class="mb-4">
                        <label class="block text-sm font-medium text-gray-700">Add a task:</label>
                        <div class="flex items-center space-x-2 mt-1">
                            <input type="text" placeholder="e.g., Review Chapter 5" class="task-input flex-1 px-4 py-2 border rounded-md focus:ring focus:ring-purple-200 focus:outline-none">
                            <button class="add-task-btn bg-purple-500 text-white p-2 rounded-full hover:bg-purple-600 transition-colors duration-200" data-subject-index="${subjectIndex}">
                                <svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5" viewBox="0 0 20 20" fill="currentColor">
                                  <path fill-rule="evenodd" d="M10 5a1 1 0 011 1v3h3a1 1 0 110 2h-3v3a1 1 0 11-2 0v-3H6a1 1 0 110-2h3V6a1 1 0 011-1z" clip-rule="evenodd" />
                                </svg>
                            </button>
                        </div>
                    </div>
                    <div class="tasks-list-container">
                        <ul class="tasks-list space-y-3"></ul>
                    </div>
                `;

                const tasksList = subjectCard.querySelector('.tasks-list');
                const sortedTasks = subject.tasks.sort((a, b) => new Date(a.date) - new Date(b.date));

                sortedTasks.forEach(task => {
                    const taskItem = document.createElement('li');
                    taskItem.className = 'flex items-center justify-between bg-gray-50 p-3 rounded-lg border border-gray-200 shadow-sm';
                    taskItem.innerHTML = `
                        <div>
                            <p class="text-gray-900">${task.description}</p>
                            <span class="text-xs text-gray-500 mt-1 block">Date: ${task.date}</span>
                        </div>
                    `;
                    tasksList.appendChild(taskItem);
                });

                subjectCard.querySelector('.add-task-btn').addEventListener('click', (e) => {
                    const taskInput = e.target.closest('.add-task-btn').previousElementSibling;
                    const subjectIndex = e.target.closest('.add-task-btn').dataset.subjectIndex;
                    addTask(subjectIndex, taskInput.value.trim());
                    taskInput.value = '';
                });

                scheduleContainer.appendChild(subjectCard);
            });
        };

        // Main app initialization function
        const initApp = async () => {
            onAuthStateChanged(auth, async (user) => {
                if (user) {
                    userId = user.uid;
                    userIdDisplay.textContent = `User ID: ${userId}`;
                    userSchedulesDocRef = doc(db, `/artifacts/${appId}/users/${userId}/schedules/my_schedule`);

                    // Set up a real-time listener for the user's schedule document
                    onSnapshot(userSchedulesDocRef, (docSnap) => {
                        if (docSnap.exists()) {
                            renderSchedule(docSnap.data());
                        } else {
                            renderSchedule({ subjects: [] });
                        }
                    }, (error) => {
                        console.error("Error listening to schedule changes: ", error);
                        showMessage("Error in real-time updates. Please refresh the page.");
                    });
                } else {
                    // Sign in anonymously if there's no auth token
                    try {
                        if (initialAuthToken) {
                            await signInWithCustomToken(auth, initialAuthToken);
                        } else {
                            await signInAnonymously(auth);
                        }
                    } catch (e) {
                        console.error("Error during authentication: ", e);
                        showMessage("Authentication failed. Please check your connection and try again.");
                    }
                }
            });

            document.getElementById('addSubjectBtn').addEventListener('click', addSubject);
        };

        // Start the app when the window loads
        window.onload = initApp;
    </script>
    <style>
        body {
            font-family: 'Inter', sans-serif;
        }
        .message-box {
            position: fixed;
            top: 20px;
            left: 50%;
            transform: translateX(-50%);
            z-index: 1000;
            transition: all 0.3s ease-in-out;
        }
    </style>
</head>
<body class="bg-gray-100 min-h-screen p-4 flex flex-col items-center">

    <!-- Message Box (hidden by default) -->
    <div id="messageBox" class="message-box hidden bg-red-500 text-white px-6 py-3 rounded-lg shadow-xl flex items-center justify-between space-x-4">
        <p id="messageText" class="font-medium"></p>
        <button id="messageCloseBtn" class="bg-transparent border-none text-white opacity-70 hover:opacity-100 transition-opacity duration-200">
            <svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5" viewBox="0 0 20 20" fill="currentColor">
                <path fill-rule="evenodd" d="M4.293 4.293a1 1 0 011.414 0L10 8.586l4.293-4.293a1 1 0 111.414 1.414L11.414 10l4.293 4.293a1 1 0 01-1.414 1.414L10 11.414l-4.293 4.293a1 1 0 01-1.414-1.414L8.586 10 4.293 5.707a1 1 0 010-1.414z" clip-rule="evenodd" />
            </svg>
        </button>
    </div>

    <!-- Header & User ID Display -->
    <header class="w-full max-w-4xl text-center mb-8">
        <h1 class="text-4xl font-extrabold text-gray-900 mb-2">Exam Schedule Planner</h1>
        <p class="text-gray-600 mb-4">Organize your study plan and prepare for success!</p>
        <div class="bg-blue-100 text-blue-800 text-xs font-semibold px-2.5 py-0.5 rounded-full inline-block mt-2">
            <span id="userIdDisplay">Loading User ID...</span>
        </div>
    </header>

    <!-- Main Content Area -->
    <main class="w-full max-w-4xl space-y-8">
        <!-- Input Form to Add Subjects -->
        <section class="bg-white p-6 rounded-lg shadow-md border border-gray-200">
            <h2 class="text-2xl font-semibold text-gray-800 mb-4">Add a New Subject</h2>
            <div class="grid grid-cols-1 md:grid-cols-2 gap-4">
                <input type="text" id="subjectInput" placeholder="Enter subject name (e.g., Physics)" class="px-4 py-2 border rounded-md focus:ring focus:ring-purple-200 focus:outline-none">
                <input type="date" id="examDateInput" class="px-4 py-2 border rounded-md focus:ring focus:ring-purple-200 focus:outline-none">
            </div>
            <button id="addSubjectBtn" class="mt-4 w-full bg-purple-600 text-white font-bold py-2 px-4 rounded-md shadow-lg hover:bg-purple-700 transition-colors duration-200 transform hover:scale-105">
                Add Subject to Schedule
            </button>
        </section>

        <!-- Schedule Display Area -->
        <section id="scheduleContainer" class="space-y-6">
            <!-- Schedule cards will be rendered here by JavaScript -->
        </section>
    </main>

</body>
</html>
