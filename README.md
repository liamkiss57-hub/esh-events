import React, { useState, useEffect, useCallback, useMemo } from 'react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from 'firebase/auth';
import { getFirestore, collection, addDoc, onSnapshot, query, deleteDoc, doc, setDoc, getDocs, deleteDoc as firestoreDeleteDoc, orderBy } from 'firebase/firestore';

// Global variables provided by the environment
const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
const firebaseConfig = JSON.parse(typeof __firebase_config !== 'undefined' ? __firebase_config : '{}');
const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;

// --- CONFIGURATION ---
const ADMIN_PIN = "1234";

// Helper to format Date
const formatDateTime = (dateObj) => {
  if (!(dateObj instanceof Date && !isNaN(dateObj))) return 'Invalid Date';
  return dateObj.toLocaleDateString('en-US', { 
    weekday: 'short', 
    year: 'numeric', 
    month: 'short', 
    day: 'numeric', 
    hour: '2-digit', 
    minute: '2-digit' 
  });
};

// Modal Component for showing attendees
const AttendeeModal = ({ eventTitle, attendeeIds, onClose }) => (
  <div className="fixed inset-0 bg-gray-900 bg-opacity-75 flex items-center justify-center p-4 z-50">
    <div className="bg-white rounded-xl shadow-2xl w-full max-w-lg p-6">
      <div className="flex justify-between items-center border-b pb-3 mb-4">
        <h3 className="text-2xl font-bold text-gray-800">Attendees for "{eventTitle}"</h3>
        <button onClick={onClose} className="text-gray-400 hover:text-gray-600">
          <svg className="w-6 h-6" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path strokeLinecap="round" strokeLinejoin="round" strokeWidth="2" d="M6 18L18 6M6 6l12 12"></path></svg>
        </button>
      </div>
      
      {attendeeIds.length === 0 ? (
        <p className="text-gray-500">No RSVPs yet!</p>
      ) : (
        <ul className="space-y-2 max-h-64 overflow-y-auto">
          {attendeeIds.map((id, index) => (
            <li key={index} className="bg-gray-100 p-2 rounded-lg text-sm font-mono text-gray-700 break-words">
              {id}
            </li>
          ))}
        </ul>
      )}
      <div className="mt-6 text-right">
        <button onClick={onClose} className="px-4 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700 transition">Close</button>
      </div>
    </div>
  </div>
);

const App = () => {
  const [db, setDb] = useState(null);
  const [auth, setAuth] = useState(null);
  const [userId, setUserId] = useState(null);
  const [events, setEvents] = useState([]);
  const [banners, setBanners] = useState([]); // State for Banner Images
  const [currentBannerIndex, setCurrentBannerIndex] = useState(0);
  
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState('');
  
  const [newEvent, setNewEvent] = useState({ title: '', description: '', dateTime: '' });
  const [newBannerUrl, setNewBannerUrl] = useState('');
  
  const [isAuthReady, setIsAuthReady] = useState(false);
  const [isFetchingData, setIsFetchingData] = useState(false);
  const [modalData, setModalData] = useState(null);
  
  // --- Admin State ---
  const [isAdmin, setIsAdmin] = useState(false);
  const [pinInput, setPinInput] = useState("");
  const [showAdminLogin, setShowAdminLogin] = useState(false);
  const [adminTab, setAdminTab] = useState('events'); // 'events' or 'banners'
  const [logoUrl, setLogoUrl] = useState(localStorage.getItem('appLogoUrl') || '');
  const [showLogoInput, setShowLogoInput] = useState(false);

  // Firestore path constants
  const getEventsPath = useCallback(() => `artifacts/${appId}/public/data/events`, []);
  const getBannersPath = useCallback(() => `artifacts/${appId}/public/data/banners`, []);

  // 1. Firebase Initialization
  useEffect(() => {
    try {
      const app = initializeApp(firebaseConfig);
      const firestore = getFirestore(app);
      const userAuth = getAuth(app);
      setDb(firestore);
      setAuth(userAuth);
      
      const unsubscribeAuth = onAuthStateChanged(userAuth, async (user) => {
        if (!user) {
          if (initialAuthToken) await signInWithCustomToken(userAuth, initialAuthToken);
          else await signInAnonymously(userAuth);
        }
        setUserId(userAuth.currentUser?.uid || 'anonymous');
        setIsAuthReady(true);
        setLoading(false);
      });
      return () => unsubscribeAuth();
    } catch (e) {
      console.error("Firebase Initialization Error: ", e);
      setError("Failed to initialize Firebase.");
      setLoading(false);
    }
  }, []);

  // 2. Data Fetching (Events & Banners)
  useEffect(() => {
    if (!db || !userId || !isAuthReady) return;

    setIsFetchingData(true);
    
    // -- Fetch Events --
    const eventsQuery = query(collection(db, getEventsPath()));
    const unsubscribeEvents = onSnapshot(eventsQuery, async (snapshot) => {
        const eventPromises = snapshot.docs.map(async (docSnapshot) => {
            const eventData = docSnapshot.data();
            const eventId = docSnapshot.id;
            const rsvpCollectionRef = collection(db, getEventsPath(), eventId, 'rsvps');
            const rsvpSnapshot = await getDocs(rsvpCollectionRef);
            const attendeeIds = rsvpSnapshot.docs.map(rsvpDoc => rsvpDoc.data().userId); 
            const isAttending = attendeeIds.includes(userId);

            return {
                id: eventId,
                ...eventData,
                dateObj: new Date(eventData.dateTime), 
                attendeeCount: attendeeIds.length,
                attendeeIds: attendeeIds,
                isAttending: isAttending,
            };
        });

        const fetchedEvents = await Promise.all(eventPromises);
        const validEvents = fetchedEvents.filter(event => event.dateObj instanceof Date && !isNaN(event.dateObj));
        validEvents.sort((a, b) => a.dateObj.getTime() - b.dateObj.getTime());
        setEvents(validEvents);
        setIsFetchingData(false);
    });

    // -- Fetch Banners --
    const bannersQuery = query(collection(db, getBannersPath()), orderBy('createdAt', 'desc'));
    const unsubscribeBanners = onSnapshot(bannersQuery, (snapshot) => {
        const fetchedBanners = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
        setBanners(fetchedBanners);
    });

    return () => {
        unsubscribeEvents();
        unsubscribeBanners();
    };
  }, [db, userId, isAuthReady, getEventsPath, getBannersPath]);

  // 3. Carousel Logic
  useEffect(() => {
      if (banners.length <= 1) return;
      const interval = setInterval(() => {
          setCurrentBannerIndex((prev) => (prev + 1) % banners.length);
      }, 5000); // 5 seconds per slide
      return () => clearInterval(interval);
  }, [banners.length]);

  // 4. Notifications Logic
  const imminentEvents = useMemo(() => {
    const now = new Date();
    const fortyEightHours = 48 * 60 * 60 * 1000;
    const futureLimit = now.getTime() + fortyEightHours;
    return events.filter(event => {
      const eventTime = event.dateObj.getTime();
      return eventTime > now.getTime() && eventTime < futureLimit;
    });
  }, [events]);

  // 5. Actions (Events)
  const handleInputChange = (e) => setNewEvent({ ...newEvent, [e.target.name]: e.target.value });
  
  const handleAddEvent = async (e) => {
    e.preventDefault();
    if (!isAdmin || !db || !userId) return;
    try {
      await addDoc(collection(db, getEventsPath()), {
        title: newEvent.title,
        description: newEvent.description,
        dateTime: newEvent.dateTime, 
        ownerId: userId,
        createdAt: new Date().toISOString(),
      });
      setNewEvent({ title: '', description: '', dateTime: '' });
    } catch (e) { setError("Failed to add event."); }
  };

  const handleDeleteEvent = async (eventId) => {
    if (!isAdmin || !db) return;
    if (!window.confirm("Delete this event?")) return;
    try { await firestoreDeleteDoc(doc(db, getEventsPath(), eventId)); } 
    catch (e) { setError("Failed to delete event."); }
  };

  const handleRsvpToggle = async (event) => {
    if (!db || !userId) return;
    const rsvpDocRef = doc(db, getEventsPath(), event.id, 'rsvps', userId);
    try {
      if (event.isAttending) await firestoreDeleteDoc(rsvpDocRef);
      else await setDoc(rsvpDocRef, { userId: userId, timestamp: new Date().toISOString() });
    } catch (e) { setError("Failed to update RSVP."); }
  };

  // 6. Actions (Banners)
  const handleAddBanner = async (e) => {
      e.preventDefault();
      if (!isAdmin || !newBannerUrl || !db) return;
      try {
          await addDoc(collection(db, getBannersPath()), {
              url: newBannerUrl,
              createdAt: new Date().toISOString()
          });
          setNewBannerUrl('');
      } catch (e) { setError("Failed to add banner."); }
  };

  const handleDeleteBanner = async (id) => {
      if (!isAdmin || !db) return;
      if (!window.confirm("Remove this banner?")) return;
      try { await firestoreDeleteDoc(doc(db, getBannersPath(), id)); }
      catch (e) { setError("Failed to remove banner."); }
  };

  // 7. Admin Auth
  const handleAdminLogin = (e) => {
      e.preventDefault();
      if (pinInput === ADMIN_PIN) {
          setIsAdmin(true); setShowAdminLogin(false); setPinInput("");
      } else { alert("Incorrect PIN"); }
  };

  const handleLogoUrlChange = (e) => {
      setLogoUrl(e.target.value);
      localStorage.setItem('appLogoUrl', e.target.value);
  };
  
  const handleShareApp = () => {
    navigator.clipboard.writeText(window.location.href)
      .then(() => alert("Copied to clipboard!"))
      .catch(() => alert("Failed to copy."));
  };

  if (loading) return <div className="flex justify-center items-center h-screen bg-blue-900 text-white">Loading...</div>;

  return (
    <div className="min-h-screen bg-blue-900 font-inter pb-12">
      
      {/* --- CAROUSEL SECTION --- */}
      {banners.length > 0 && (
          <div className="w-full h-48 sm:h-64 md:h-80 bg-black relative overflow-hidden mb-6 group">
              {banners.map((banner, index) => (
                  <div 
                    key={banner.id}
                    className={`absolute inset-0 transition-opacity duration-1000 ease-in-out ${index === currentBannerIndex ? 'opacity-100' : 'opacity-0'}`}
                  >
                      <img src={banner.url} alt="Banner" className="w-full h-full object-cover" />
                  </div>
              ))}
              
              {/* Carousel Controls (dots) */}
              <div className="absolute bottom-4 left-0 right-0 flex justify-center space-x-2">
                  {banners.map((_, idx) => (
                      <button 
                        key={idx}
                        onClick={() => setCurrentBannerIndex(idx)}
                        className={`w-2 h-2 rounded-full transition-all ${idx === currentBannerIndex ? 'bg-white w-4' : 'bg-white/50'}`}
                      />
                  ))}
              </div>
          </div>
      )}

      {/* --- HEADER --- */}
      <header className="px-4 sm:px-8 mb-8 flex flex-col items-center relative">
        {/* Admin Buttons */}
        <div className="absolute top-0 right-4 flex space-x-2">
            {isAdmin ? (
                <button onClick={() => setIsAdmin(false)} className="text-red-300 hover:text-white text-xs uppercase font-bold">Logout</button>
            ) : (
                <button onClick={() => setShowAdminLogin(!showAdminLogin)} className="text-blue-300 hover:text-white text-xs uppercase font-bold">Owner Login</button>
            )}
        </div>

        {/* Logo */}
        <div className="mb-4 mt-4 relative group">
            {logoUrl ? (
                <img src={logoUrl} alt="App Logo" className="h-24 w-auto object-contain rounded-md bg-white p-2 shadow-lg" />
            ) : (
                <div className="h-20 w-20 bg-blue-800 rounded-md flex items-center justify-center text-blue-300 shadow-lg">Logo</div>
            )}
            {isAdmin && (
                <button onClick={() => setShowLogoInput(!showLogoInput)} className="absolute -bottom-2 -right-2 bg-gray-200 hover:bg-gray-300 text-gray-700 rounded-full p-1 shadow-sm">
                    <svg className="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path strokeLinecap="round" strokeLinejoin="round" strokeWidth="2" d="M15.232 5.232l3.536 3.536m-2.036-5.036a2.5 2.5 0 113.536 3.536L6.5 21.036H3v-3.572L16.732 3.732z"></path></svg>
                </button>
            )}
        </div>
        
        {isAdmin && showLogoInput && (
            <input type="text" placeholder="Logo URL..." value={logoUrl} onChange={handleLogoUrlChange} className="mb-4 w-full max-w-xs px-2 py-1 border rounded text-sm" />
        )}

        <h1 className="text-3xl sm:text-4xl font-extrabold text-white mb-2 text-center">Event Hub</h1>
        <button onClick={handleShareApp} className="px-4 py-2 bg-blue-700 text-white rounded-full hover:bg-blue-600 transition text-sm font-bold shadow-md flex items-center">
            <svg className="w-4 h-4 mr-2" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path strokeLinecap="round" strokeLinejoin="round" strokeWidth="2" d="M8.684 13.342C8.886 12.938 9 12.482 9 12c0-.482-.114-.938-.316-1.342m0 2.684a3 3 0 110-2.684m0 2.684l6.632 3.316m-6.632-6l6.632-3.316m0 0a3 3 0 105.367-2.684 3 3 0 00-5.367 2.684zm0 9.316a3 3 0 105.368 2.684 3 3 0 00-5.368-2.684z"></path></svg>
            Share App Link
        </button>
      </header>

      {/* --- NOTIFICATIONS --- */}
      {imminentEvents.length > 0 && (
        <div className="mx-4 sm:mx-auto max-w-3xl bg-yellow-100 border-l-4 border-yellow-500 text-yellow-800 p-4 mb-8 rounded-lg shadow-md">
          <p className="font-bold flex items-center"><span className="mr-2">⚠️</span> Happening Soon!</p>
          <ul className="list-disc list-inside mt-2 space-y-1">
            {imminentEvents.map(ev => <li key={ev.id} className="text-sm">"{ev.title}" - {formatDateTime(ev.dateObj)}</li>)}
          </ul>
        </div>
      )}

      {/* --- ADMIN PANEL --- */}
      {isAdmin && (
        <section className="mx-4 sm:mx-auto max-w-3xl bg-white rounded-xl shadow-lg mb-8 border-2 border-blue-400 overflow-hidden">
            <div className="bg-blue-100 p-2 flex space-x-1">
                <button onClick={() => setAdminTab('events')} className={`flex-1 py-2 text-sm font-bold rounded ${adminTab === 'events' ? 'bg-white text-blue-800 shadow-sm' : 'text-blue-600 hover:bg-blue-50'}`}>Manage Events</button>
                <button onClick={() => setAdminTab('banners')} className={`flex-1 py-2 text-sm font-bold rounded ${adminTab === 'banners' ? 'bg-white text-blue-800 shadow-sm' : 'text-blue-600 hover:bg-blue-50'}`}>Manage Banners</button>
            </div>

            <div className="p-6">
                {/* Event Form */}
                {adminTab === 'events' && (
                    <form onSubmit={handleAddEvent} className="space-y-4">
                        <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
                            <input type="text" name="title" value={newEvent.title} onChange={handleInputChange} required placeholder="Event Title" className="block w-full px-3 py-2 border rounded-lg" />
                            <input type="datetime-local" name="dateTime" value={newEvent.dateTime} onChange={handleInputChange} required className="block w-full px-3 py-2 border rounded-lg" />
                        </div>
                        <textarea name="description" value={newEvent.description} onChange={handleInputChange} rows="2" placeholder="Description" className="block w-full px-3 py-2 border rounded-lg" />
                        <button type="submit" className="w-full py-2 bg-blue-600 text-white font-bold rounded-lg hover:bg-blue-700">Add Event</button>
                    </form>
                )}

                {/* Banner Form */}
                {adminTab === 'banners' && (
                    <div>
                        <form onSubmit={handleAddBanner} className="flex gap-2 mb-4">
                            <input type="text" value={newBannerUrl} onChange={(e) => setNewBannerUrl(e.target.value)} required placeholder="Paste Image URL here..." className="flex-1 px-3 py-2 border rounded-lg" />
                            <button type="submit" className="px-4 py-2 bg-blue-600 text-white font-bold rounded-lg">Add</button>
                        </form>
                        
                        <div className="space-y-2">
                            <h4 className="font-semibold text-gray-600">Active Banners:</h4>
                            {banners.length === 0 && <p className="text-gray-400 text-sm">No banners active.</p>}
                            {banners.map(banner => (
                                <div key={banner.id} className="flex items-center justify-between bg-gray-50 p-2 rounded border">
                                    <div className="flex items-center space-x-3">
                                        <img src={banner.url} alt="Thumbnail" className="w-12 h-8 object-cover rounded bg-gray-200" />
                                        <span className="text-xs text-gray-500 truncate max-w-[150px]">{banner.url}</span>
                                    </div>
                                    <button onClick={() => handleDeleteBanner(banner.id)} className="text-red-500 hover:text-red-700 text-sm font-bold">Remove</button>
                                </div>
                            ))}
                        </div>
                    </div>
                )}
            </div>
        </section>
      )}

      {/* --- PUBLIC EVENT LIST --- */}
      <section className="mx-4 sm:mx-auto max-w-3xl space-y-4">
        <h2 className="text-2xl font-bold text-white mb-4">Upcoming Events</h2>
        {events.length === 0 ? (
            <div className="p-8 text-center text-gray-500 bg-white rounded-xl">No events scheduled.</div>
        ) : (
            events.map(event => (
              <div key={event.id} className="bg-white p-5 rounded-xl shadow-md flex flex-col sm:flex-row gap-4 hover:shadow-lg transition">
                <div className="flex-1">
                  <p className="text-xs font-bold uppercase text-blue-600 mb-1">{formatDateTime(event.dateObj)}</p>
                  <h3 className="text-xl font-bold text-gray-900">{event.title}</h3>
                  <p className="text-gray-600 text-sm mt-1">{event.description}</p>
                </div>
                <div className="flex flex-col sm:flex-row items-start sm:items-center gap-2">
                    <button onClick={() => isAdmin ? setModalData({ eventTitle: event.title, attendeeIds: event.attendeeIds }) : null} className={`text-sm px-3 py-1 rounded-full ${isAdmin ? 'bg-green-100 text-green-800' : 'bg-gray-100 text-gray-600 cursor-default'}`}>
                        {event.attendeeCount} Going
                    </button>
                    <button onClick={() => handleRsvpToggle(event)} className={`w-full sm:w-auto px-4 py-2 text-sm font-bold rounded-lg border-2 ${event.isAttending ? 'border-red-500 text-red-500 hover:bg-red-50' : 'border-transparent bg-blue-600 text-white hover:bg-blue-700'}`}>
                      {event.isAttending ? 'Cancel RSVP' : 'RSVP Now'}
                    </button>
                    {isAdmin && (
                        <button onClick={() => handleDeleteEvent(event.id)} className="text-gray-400 hover:text-red-600 p-2"><svg className="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path strokeLinecap="round" strokeLinejoin="round" strokeWidth="2" d="M19 7l-.867 12.142A2 2 0 0116.138 21H7.862a2 2 0 01-1.995-1.858L5 7m5 4v6m4-6v6m1-10V4a1 1 0 00-1-1h-4a1 1 0 00-1 1v3M4 7h16"></path></svg></button>
                    )}
                <
