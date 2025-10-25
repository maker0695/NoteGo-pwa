// FILE: NoteGoScheduler.jsx
// APP NAME: NoteGo - The Logbook Schedule & Team Manager
// FRAMEWORK: React, with Firebase Integration (Authentication, Firestore for Data & Chat)
// STYLES: Tailwind CSS for Notebook Aesthetic

import React, { useState, useEffect, createContext, useContext } from 'react';
import { initializeApp } from 'firebase/app';
import { getFirestore, collection, getDocs, doc, setDoc, query, orderBy, onSnapshot, addDoc, serverTimestamp } from 'firebase/firestore';
import { getAuth, signInAnonymously, onAuthStateChanged } from 'firebase/auth';

// ----------------------------------------------------------------------
// 1. FIREBASE CONFIG & INITIALIZATION
// NOTE: REPLACE THESE WITH YOUR OWN FIREBASE PROJECT CREDENTIALS
// ----------------------------------------------------------------------
const firebaseConfig = {
  apiKey: "YOUR_API_KEY",
  authDomain: "YOUR_PROJECT_ID.firebaseapp.com",
  projectId: "YOUR_PROJECT_ID",
  storageBucket: "YOUR_PROJECT_ID.appspot.com",
  messagingSenderId: "YOUR_MESSAGING_SENDER_ID",
  appId: "YOUR_APP_ID"
};

const app = initializeApp(firebaseConfig);
const db = getFirestore(app);
const auth = getAuth(app);

// ----------------------------------------------------------------------
// 2. CONTEXT & AUTH PROVIDER
// ----------------------------------------------------------------------
const AppContext = createContext();

const AppProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [schedules, setSchedules] = useState([]);

  useEffect(() => {
    // 1. Handle Anonymous User Authentication for demo purposes
    onAuthStateChanged(auth, (currentUser) => {
      if (!currentUser) {
        signInAnonymously(auth).then(userCredential => {
          setUser(userCredential.user);
        }).catch(console.error);
      } else {
        setUser(currentUser);
      }
    });

    // 2. Real-time Schedule Listener (Core Scheduling)
    if (user) {
      const q = query(collection(db, "schedules"), orderBy("startTime", "asc"));
      const unsubscribe = onSnapshot(q, (snapshot) => {
        const scheduleList = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
        setSchedules(scheduleList);
      });
      return () => unsubscribe();
    }
  }, [user]);

  // Function to create a new job schedule
  const createSchedule = async (scheduleData) => {
    try {
      await addDoc(collection(db, "schedules"), {
        ...scheduleData,
        createdAt: serverTimestamp(),
        // Simulates notification sent to workers
        notificationsSent: true 
      });
      alert("Shift created! Notifications simulated.");
    } catch (e) {
      console.error("Error adding document: ", e);
    }
  };

  return (
    <AppContext.Provider value={{ user, schedules, createSchedule, db }}>
      {children}
    </AppContext.Provider>
  );
};

// ----------------------------------------------------------------------
// 3. UI COMPONENTS
// ----------------------------------------------------------------------

// ** Aesthetic Helper: Notebook Style **
const NotebookPaper = ({ children, className = '' }) => (
  <div className={`bg-amber-50 border-r-4 border-gray-400 p-6 shadow-xl ${className}`} style={{
    backgroundImage: `linear-gradient(to bottom, #d6d6d6 1px, transparent 1px), linear-gradient(to right, #d6d6d6 1px, transparent 1px)`,
    backgroundSize: `100% 30px, 30px 100%`, // Horizontal lines every 30px
    backgroundPosition: `0 10px`, // Start line 10px down
  }}>
    {children}
  </div>
);

// ** Feature: Job-Specific Chat **
const JobChat = ({ jobId }) => {
  const { user, db } = useContext(AppContext);
  const [messages, setMessages] = useState([]);
  const [newMessage, setNewMessage] = useState('');

  useEffect(() => {
    if (!jobId) return;
    const q = query(collection(db, `jobs/${jobId}/chat`), orderBy("timestamp", "asc"));
    const unsubscribe = onSnapshot(q, (snapshot) => {
      setMessages(snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() })));
    });
    return () => unsubscribe();
  }, [jobId, db]);

  const sendMessage = async (e) => {
    e.preventDefault();
    if (newMessage.trim() === '') return;
    await addDoc(collection(db, `jobs/${jobId}/chat`), {
      text: newMessage,
      timestamp: serverTimestamp(),
      userId: user.uid,
      userName: user.uid.substring(0, 4) // Simplified user name
    });
    setNewMessage('');
  };

  return (
    <div className="mt-4 border-t border-gray-300 pt-4">
      <h4 className="font-bold text-lg text-indigo-700">💬 Job Chat for #{jobId}</h4>
      <div className="h-40 overflow-y-scroll p-2 bg-white border border-gray-200">
        {messages.map((msg) => (
          <div key={msg.id} className={`text-sm my-1 ${msg.userId === user.uid ? 'text-right' : 'text-left'}`}>
            <span className={`inline-block p-2 rounded-lg ${msg.userId === user.uid ? 'bg-indigo-200' : 'bg-gray-200'}`}>
              <span className="font-semibold">{msg.userName}:</span> {msg.text}
            </span>
          </div>
        ))}
      </div>
      <form onSubmit={sendMessage} className="flex mt-2">
        <input
          type="text"
          value={newMessage}
          onChange={(e) => setNewMessage(e.target.value)}
          placeholder="Type message..."
          className="flex-grow p-2 border border-gray-400"
        />
        <button type="submit" className="ml-2 px-4 py-2 bg-indigo-600 text-white hover:bg-indigo-700">Send</button>
      </form>
    </div>
  );
};

// ** Feature: Schedule Creation Form **
const AdminScheduler = () => {
  const { createSchedule } = useContext(AppContext);
  const [title, setTitle] = useState('');
  const [description, setDescription] = useState('');
  const [worker, setWorker] = useState('Worker A');
  const [startTime, setStartTime] = useState('');
  const [endTime, setEndTime] = useState('');

  const handleSubmit = (e) => {
    e.preventDefault();
    const newJobId = Math.random().toString(36).substr(2, 9).toUpperCase();

    createSchedule({
      jobId: newJobId,
      title,
      description,
      assignedWorker: worker,
      startTime: new Date(startTime).getTime(),
      endTime: new Date(endTime).getTime(),
      status: 'Scheduled',
    });

    // Reset form
    setTitle('');
    setDescription('');
    setStartTime('');
    setEndTime('');
  };

  return (
    <NotebookPaper className="mb-8">
      <h3 className="text-xl font-bold mb-4 text-indigo-800 border-b pb-2">Admin: Create New Shift</h3>
      <form onSubmit={handleSubmit} className="space-y-3">
        <input type="text" placeholder="Job Title" value={title} onChange={(e) => setTitle(e.target.value)} required className="w-full p-2 border border-gray-400 bg-white" />
        <textarea placeholder="Brief Description (for worker)" value={description} onChange={(e) => setDescription(e.target.value)} required className="w-full p-2 border border-gray-400 bg-white h-20"></textarea>
        <select value={worker} onChange={(e) => setWorker(e.target.value)} className="w-full p-2 border border-gray-400 bg-white">
          <option value="Worker A">Worker A</option>
          <option value="Worker B">Worker B</option>
          <option value="Worker C">Worker C</option>
        </select>
        <label className="block text-sm">Start Time:</label>
        <input type="datetime-local" value={startTime} onChange={(e) => setStartTime(e.target.value)} required className="w-full p-2 border border-gray-400 bg-white" />
        <label className="block text-sm">End Time:</label>
        <input type="datetime-local" value={endTime} onChange={(e) => setEndTime(e.target.value)} required className="w-full p-2 border border-gray-400 bg-white" />
        <button type="submit" className="w-full p-3 bg-indigo-600 text-white font-semibold hover:bg-indigo-700 mt-4">Publish Shift & Notify</button>
      </form>
    </NotebookPaper>
  );
};

// ** Main Worker Schedule View (Worker A's perspective for demo) **
const WorkerScheduleView = () => {
  const { schedules } = useContext(AppContext);

  return (
    <div className="p-4">
      <h3 className="text-2xl font-bold mb-4 text-green-700">Worker A's Schedule: My Shifts</h3>
      {schedules.filter(s => s.assignedWorker === 'Worker A').map(schedule => (
        <NotebookPaper key={schedule.id} className="mb-6 border-l-8 border-green-500">
          <p className="text-sm text-gray-500">Job ID: {schedule.jobId}</p>
          <h4 className="text-xl font-bold text-indigo-700">{schedule.title}</h4>
          <p className="mt-1">**Brief:** {schedule.description}</p>
          <p className="mt-2 text-sm">**Time:** {new Date(schedule.startTime).toLocaleString()} - {new Date(schedule.endTime).toLocaleTimeString()}</p>
          <p className="text-sm">**Worker:** {schedule.assignedWorker}</p>

          {/* Feature: Duty Checklist (Simplified) */}
          <div className="mt-4 pt-2 border-t border-gray-300">
            <h5 className="font-semibold">Duties Checklist</h5>
            <label className="flex items-center space-x-2"><input type="checkbox" className="form-checkbox text-green-600" /> <span>Check tools & vehicle</span></label>
            <label className="flex items-center space-x-2"><input type="checkbox" className="form-checkbox text-green-600" /> <span>Complete main task</span></label>
            <label className="flex items-center space-x-2"><input type="checkbox" className="form-checkbox text-green-600" /> <span>Clean up site</span></label>
          </div>
          
          {/* Feature: Job-Specific Chat Integration */}
          <JobChat jobId={schedule.jobId} />
        </NotebookPaper>
      ))}
    </div>
  );
};

// ** Main Application Layout **
const NoteGoApp = () => {
  const [isWorkerView, setIsWorkerView] = useState(true);

  return (
    <AppProvider>
      <div className="min-h-screen bg-gray-100 p-8">
        <header className="bg-indigo-600 text-white p-4 flex justify-between items-center shadow-lg">
          <h1 className="text-3xl font-serif tracking-wider">📒 **NoteGo** Scheduler</h1>
          <button
            onClick={() => setIsWorkerView(!isWorkerView)}
            className="px-4 py-2 bg-white text-indigo-600 rounded-full font-semibold hover:bg-indigo-100 transition duration-150"
          >
            Switch to {isWorkerView ? 'Admin View' : 'Worker View'}
          </button>
        </header>

        <div className="mt-8 max-w-4xl mx-auto border-4 border-indigo-900 rounded-lg overflow-hidden bg-gray-50">
          {isWorkerView ? <WorkerScheduleView /> : <AdminScheduler />}
        </div>
        
        {/* Placeholder for Separate Notes Section */}
        <NotebookPaper className="mt-8 max-w-4xl mx-auto">
             <h3 className="text-xl font-bold text-gray-600">📝 Notes Section (Placeholder)</h3>
             <textarea placeholder="Jot down personal reminders or ideas here..." className="w-full p-2 border border-gray-300 h-20 mt-2"></textarea>
        </NotebookPaper>

      </div>
    </AppProvider>
  );
};

export default NoteGoApp;
