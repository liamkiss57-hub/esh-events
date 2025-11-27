import React, { useState, useEffect, useCallback, useMemo } from 'react';
import { initializeApp } from 'firebase/app';
import {
  getAuth,
  signInAnonymously,
  signInWithCustomToken,
  onAuthStateChanged,
} from 'firebase/auth';
import {
  getFirestore,
  collection,
  addDoc,
  onSnapshot,
  query,
  deleteDoc,
  doc,
  setDoc,
  getDocs,
  orderBy,
} from 'firebase/firestore';

// === CONFIGURATION (Replace these in your hosting environment) ===
const firebaseConfig = {
  apiKey: import.meta.env.VITE_FIREBASE_API_KEY || "your-api-key",
  authDomain: import.meta.env.VITE_FIREBASE_AUTH_DOMAIN || "your-app.firebaseapp.com",
  projectId: import.meta.env.VITE_FIREBASE_PROJECT_ID || "your-project-id",
  storageBucket: import.meta.env.VITE_FIREBASE_STORAGE_BUCKET || "your-app.appspot.com",
  messagingSenderId: import.meta.env.VITE_FIREBASE_MESSAGING_SENDER_ID || "123456789",
  appId: import.meta.env.VITE_FIREBASE_APP_ID || "1:123456789:web:abcdef123456",
};

// Global overrides (used when deployed via your custom deployer)
const appId = typeof window.__app_id !== 'undefined' ? window.__app_id : 'default';
const initialAuthToken = typeof window.__initial_auth_token !== 'undefined' ? window.__initial_auth_token : null;

const ADMIN_PIN = "1234"; // WARNING: Change this in production!

// Helper
const formatDateTime = (dateObj) => {
  if (!(dateObj instanceof Date) || isNaN(dateObj)) return 'Invalid Date';
  return dateObj.toLocaleString('en-US', {
    weekday: 'short',
    year: 'numeric',
    month: 'short',
    day: 'numeric',
    hour: '2-digit',
    minute: '2-digit',
  });
};

// Modal Component
const AttendeeModal = ({ eventTitle, attendeeIds, onClose }) => (
  <div className="fixed inset-0 bg-black bg-opacity-60 flex items-center justify-center p-4 z-50">
    <div className="bg-white rounded-xl shadow-2xl max-w-lg w-full p-6">
      <div className="flex justify-between items-center border-b pb-3 mb-4">
        <h3 className="text-xl font-bold">Attendees: "{eventTitle}"</h3>
        <button onClick={onClose} className="text-gray-500 hover:text-gray-700">
          <svg className="w-6 h-6" fill="none" stroke="currentColor" viewBox="0 0 24 24">
            <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M6 18L18 6M6 6l12 12" />
          </svg>
        </button>
      </div>
      {attendeeIds.length === 0 ? (
        <p className="text-gray-500">No one has RSVP'd yet.</p>
      ) : (
        <ul className="space-y-2 max-h-64 overflow-y-auto">
          {attendeeIds.map((id, i) => (
            <li key={i} className="bg-gray-100 p-3 rounded text-sm font-mono break-all">
              {id}
            </li>
          ))}
        </ul>
      )}
      <button onClick={onClose} className="mt-6 w-full py-2 bg-blue-600 text-white rounded hover:bg-blue-700">
        Close
      </button>
    </div>
  </div>
);

export default function App() {
  const [db, setDb] = useState(null);
  const [auth, setAuth] = useState(null);
  const [userId, setUserId] = useState(null);
  const [events, setEvents] = useState([]);
  const [banners, setBanners] = useState([]);
  const [currentBannerIndex, setCurrentBannerIndex] = useState(0);
  const [loading, setLoading] = useState(true);
  const [modalData, setModalData] = useState(null);

  // Admin
  const [isAdmin, setIsAdmin] = useState(false);
  const [pinInput, setPinInput] = useState('');
  const [showAdminLogin, setShowAdminLogin] = useState(false);
  const [adminTab, setAdminTab] = useState('events');
  const [logoUrl, setLogoUrl] = useState(localStorage.getItem('appLogoUrl') || '');

  // Forms
  const [newEvent, setNewEvent] = useState({ title: '', description: '', dateTime: '' });
  const [newBannerUrl, setNewBannerUrl] = useState('');

  const getEventsPath = useCallback(() => `artifacts/${appId}/public/data/events`, [appId]);
  const getBannersPath = useCallback(() => `artifacts/${appId}/public/data/banners`, [appId]);

  // Firebase Init
  useEffect(() => {
    const app = initializeApp(firebaseConfig);
    const authInstance = getAuth(app);
    const dbInstance = getFirestore(app);

    setAuth(authInstance);
    setDb(dbInstance);

    const unsubscribe = onAuthStateChanged(authInstance, async (user) => {
      try {
        if (!user) {
          if (initialAuthToken) await signInWithCustomToken(authInstance, initialAuthToken);
          else await signInAnonymously(authInstance);
        } else {
          setUserId(user.uid);
          setLoading(false);
        }
      } catch (err) {
        alert("Authentication error: " + err.message);
        setLoading(false);
      }
    });

    return unsubscribe;
  }, []);

  // Fetch Events + Banners
  useEffect(() => {
    if (!db || !userId) return;

    const eventsQuery = query(collection(db, getEventsPath()));
    const unsubEvents = onSnapshot(eventsQuery, async (snapshot) => {
      const eventList = await Promise.all(
        snapshot.docs.map(async (d) => {
          const data = d.data();
          const rsvpSnap = await getDocs(collection(db, getEventsPath(), d.id, 'rsvps'));
          const attendeeIds = rsvpSnap.docs.map((r) => r.data().userId);
          let dateObj = new Date(data.dateTime);
          if (isNaN(dateObj)) dateObj = new Date();
          return {
            id: d.id,
            ...data,
            dateObj,
            attendeeCount: attendeeIds.length,
            attendeeIds,
            isAttending: attendeeIds.includes(userId),
          };
        })
      );

      eventList.sort((a, b) => a.dateObj - b.dateObj);
      setEvents(eventList);
    });

    const bannersQuery = query(collection(db, getBannersPath()), orderBy('createdAt', 'desc'));
    const unsubBanners = onSnapshot(bannersQuery, (snap) => {
      setBanners(snap.docs.map((d) => ({ id: d.id, ...d.data() })));
    });

    return () => {
      unsubEvents();
      unsubBanners();
    };
  }, [db, userId, getEventsPath, getBannersPath]);

  // Banner carousel; reset index if banners changes length
  useEffect(() => {
    setCurrentBannerIndex(0); // Always reset index on banners change!
    if (banners.length <= 1) return;
    const id = setInterval(() => setCurrentBannerIndex((i) => (i + 1) % banners.length), 5000);
    return () => clearInterval(id);
  }, [banners.length]);

  // Imminent events
  const imminentEvents = useMemo(() => {
    const now = Date.now();
    const soon = now + 48 * 60 * 60 * 1000;
    return events.filter((e) => e.dateObj > now && e.dateObj < soon);
  }, [events]);

  // Handlers
  const handleRsvpToggle = async (event) => {
    try {
      if (!userId) throw new Error('Not authenticated!');
      const ref = doc(db, getEventsPath(), event.id, 'rsvps', userId);
      if (event.isAttending) await deleteDoc(ref);
      else await setDoc(ref, { userId, timestamp: new Date().toISOString() });
    } catch (err) {
      alert("Could not update RSVP: " + err.message);
    }
  };

  const handleAddEvent = async (e) => {
    e.preventDefault();
    try {
      await addDoc(collection(db, getEventsPath()), {
        ...newEvent,
        createdAt: new Date().toISOString(),
      });
      setNewEvent({ title: '', description: '', dateTime: '' });
    } catch (err) {
      alert("Error adding event: " + err.message);
    }
  };

  const handleDeleteEvent = async (id) => {
    if (window.confirm('Delete this event?')) {
      try {
        await deleteDoc(doc(db, getEventsPath(), id));
      } catch (err) {
        alert("Error deleting event: " + err.message);
      }
    }
  };

  const handleAddBanner = async (e) => {
    e.preventDefault();
    try {
      await addDoc(collection(db, getBannersPath()), {
        url: newBannerUrl,
        createdAt: new Date().toISOString(),
      });
      setNewBannerUrl('');
    } catch (err) {
      alert("Error adding banner: " + err.message);
    }
  };

  const handleDeleteBanner = async (id) => {
    if (window.confirm('Remove banner?')) {
      try {
        await deleteDoc(doc(db, getBannersPath(), id));
      } catch (err) {
        alert("Error deleting banner: " + err.message);
      }
    }
  };

  if (loading) return <div className="flex items-center justify-center h-screen bg-blue-900 text-white text-2xl">Loading...</div>;

  return (
    <>
      {/* Banner Carousel */}
      {banners.length > 0 && (
        <div className="relative h-64 md:h-96 overflow-hidden">
          {banners.map((b, i) => (
            <img
              key={b.id}
              src={b.url}
              alt="Banner"
              className={`absolute inset-0 w-full h-full object-cover transition-opacity duration-1000 ${
                i === currentBannerIndex ? 'opacity-100' : 'opacity-0'
              }`}
            />
          ))}
          <div className="absolute bottom-4 left-1/2 -translate-x-1/2 flex gap-2">
            {banners.map((_, i) => (
              <button
                key={i}
                onClick={() => setCurrentBannerIndex(i)}
                className={`w-3 h-3 rounded-full transition-all ${
                  i === currentBannerIndex ? 'bg-white w-8' : 'bg-white/60'
                }`}
              />
            ))}
          </div>
        </div>
      )}

      <div className="min-h-screen bg-blue-900 text-white pb-20">
        <div className="max-w-4xl mx-auto px-4 py-8">
          {/* Header */}
          <header className="text-center mb-10">
            <div className="flex justify-end mb-4">
              {isAdmin ? (
                <button onClick={() => setIsAdmin(false)} className="text-red-400 text-sm">Logout</button>
              ) : (
                <button onClick={() => setShowAdminLogin(true)} className="text-blue-300 text-sm">
                  Owner Login
                </button>
              )}
            </div>

            {logoUrl && <img src={logoUrl} alt="Logo" className="mx-auto h-32 object-contain mb-4" />}
            <h1 className="text-5xl font-bold mb-4">Event Hub</h1>
            <button
              onClick={() => navigator.clipboard.writeText(location.href).then(() => alert('Link copied!'))}
              className="bg-blue-700 hover:bg-blue-600 px-6 py-3 rounded-full font-semibold"
            >
              Share App
            </button>
          </header>

          {/* Imminent Events Alert */}
          {imminentEvents.length > 0 && (
            <div className="bg-yellow-100 text-yellow-900 p-4 rounded-lg mb-8">
              <strong>Happening Soon!</strong>
              <ul className="mt-2">
                {imminentEvents.map((e) => (
                  <li key={e.id}>• {e.title} – {formatDateTime(e.dateObj)}</li>
                ))}
              </ul>
            </div>
          )}

          {/* Admin Panel */}
          {isAdmin && (
            <div className="bg-white text-gray-800 rounded-xl p-6 mb-10 shadow-xl">
              <div className="flex gap-2 mb-6">
                <button
                  onClick={() => setAdminTab('events')}
                  className={`flex-1 py-3 rounded ${adminTab === 'events' ? 'bg-blue-600 text-white' : 'bg-gray-200'}`}
                >
                  Events
                </button>
                <button
                  onClick={() => setAdminTab('banners')}
                  className={`flex-1 py-3 rounded ${adminTab === 'banners' ? 'bg-blue-600 text-white' : 'bg-gray-200'}`}
                >
                  Banners
                </button>
              </div>

              {adminTab === 'events' && (
                <form onSubmit={handleAddEvent} className="space-y-4">
                  <input
                    required
                    type="text"
                    placeholder="Title"
                    value={newEvent.title}
                    onChange={(e) => setNewEvent({ ...newEvent, title: e.target.value })}
                    className="w-full px-4 py-2 border rounded text-gray-800"
                  />
                  <input
                    required
                    type="datetime-local"
                    value={newEvent.dateTime}
                    onChange={(e) => setNewEvent({ ...newEvent, dateTime: e.target.value })}
                    className="w-full px-4 py-2 border rounded text-gray-800"
                  />
                  <textarea
                    placeholder="Description (optional)"
                    value={newEvent.description}
                    onChange={(e) => setNewEvent({ ...newEvent, description: e.target.value })}
                    className="w-full px-4 py-2 border rounded text-gray-800"
                  />
                  <button type="submit" className="w-full bg-blue-600 text-white py-3 rounded font-bold">
                    Add Event
                  </button>
                </form>
              )}

              {adminTab === 'banners' && (
                <div>
                  <form onSubmit={handleAddBanner} className="flex gap-2 mb-6">
                    <input
                      required
                      type="url"
                      placeholder="Image URL"
                      value={newBannerUrl}
                      onChange={(e) => setNewBannerUrl(e.target.value)}
                      className="flex-1 px-4 py-2 border rounded text-gray-800"
                    />
                    <button type="submit" className="bg-blue-600 text-white px-6 py-2 rounded">
                      Add
                    </button>
                  </form>
                  <div className="space-y-3">
                    {banners.map((b) => (
                      <div key={b.id} className="flex items-center justify-between bg-gray-100 p-3 rounded">
                        <img src={b.url} alt="" className="h-12 rounded" />
                        <button
                          onClick={() => handleDeleteBanner(b.id)}
                          className="text-red-600 hover:underline"
                        >
                          Remove
                        </button>
                      </div>
                    ))}
                  </div>
                </div>
              )}
            </div>
          )}

          {/* Event List */}
          <div className="space-y-6">
            {events.map((event) => (
              <div key={event.id} className="bg-white text-gray-800 rounded-xl p-6 shadow-lg">
                <div className="flex justify-between items-start">
                  <div className="flex-1">
                    <p className="text-blue-600 font-bold text-sm">{formatDateTime(event.dateObj)}</p>
                    <h3 className="text-2xl font-bold mt-1">{event.title}</h3>
                    {event.description && <p className="mt-2 text-gray-600">{event.description}</p>}
                  </div>
                  <div className="ml-4 text-right">
                    {/* Only show modal for admin */}
                    {isAdmin && (
                      <button
                        onClick={() => setModalData({ title: event.title, attendeeIds: event.attendeeIds })}
                        className="text-sm bg-green-100 text-green-800 px-3 py-1 rounded-full"
                      >
                        {event.attendeeCount} Going
                      </button>
                    )}
                    {!isAdmin && (
                      <span className="text-sm bg-green-100 text-green-800 px-3 py-1 rounded-full">
                        {event.attendeeCount} Going
                      </span>
                    )}
                  </div>
                </div>

                <div className="mt-6 flex gap-3">
                  <button
                    onClick={() => handleRsvpToggle(event)}
                    className={`flex-1 py-3 rounded font-bold ${
                      event.isAttending
                        ? 'bg-red-500 hover:bg-red-600 text-white'
                        : 'bg-blue-600 hover:bg-blue-700 text-white'
                    }`}
                  >
                    {event.isAttending ? 'Cancel RSVP' : 'RSVP Now'}
                  </button>
                  {isAdmin && (
                    <button
                      onClick={() => handleDeleteEvent(event.id)}
                      className="px-4 py-3 bg-gray-200 hover:bg-red-200 rounded"
                    >
                      Delete
                    </button>
                  )}
                </div>
              </div>
            ))}
          </div>
        </div>
      </div>

      {/* Modals */}
      {showAdminLogin && (
        <div className="fixed inset-0 bg-black bg-opacity-60 flex items-center justify-center p-4 z-50">
          <div className="bg-white rounded-xl p-8 max-w-sm w-full">
            <h3 className="text-2xl font-bold mb-4">Owner Login</h3>
            <form onSubmit={(e) => {
              e.preventDefault();
              if (pinInput === ADMIN_PIN) {
                setIsAdmin(true);
                setShowAdminLogin(false);
                setPinInput('');
              } else alert('Wrong PIN');
            }}>
              <input
                type="password"
                value={pinInput}
                onChange={(e) => setPinInput(e.target.value)}
                placeholder="Enter PIN"
                className="w-full px-4 py-3 border rounded mb-4 text-gray-800"
                autoFocus
              />
              <div className="flex gap-3">
                <button type="submit" className="flex-1 bg-blue-600 text-white py-3 rounded font-bold">
                  Login
                </button>
                <button
                  type="button"
                  onClick={() => setShowAdminLogin(false)}
                  className="flex-1 bg-gray-300 py-3 rounded font-bold"
                >
                  Cancel
                </button>
              </div>
            </form>
            <div className="mt-4 text-xs text-red-600 font-bold">
              ⚠️ Never use "1234" as a PIN in production!
            </div>
          </div>
        </div>
      )}

      {modalData && (
        <AttendeeModal
          eventTitle={modalData.title}
          attendeeIds={modalData.attendeeIds}
          onClose={() => setModalData(null)}
        />
      )}
    </>
  );
}
