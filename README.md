import React, { useState, useEffect, useMemo, useCallback } from 'react';

import { initializeApp } from 'firebase/app';

import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from 'firebase/auth';

import { getFirestore, doc, setDoc, getDoc, onSnapshot, collection, updateDoc, arrayUnion } from 'firebase/firestore';



// --- CONFIGURACIÓN DE FIREBASE ---

const appId = typeof __app_id !== 'undefined' ? __app_id : 'eco-heroe-app';

const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : null;

const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;



// Componente principal de la aplicación

const App = () => {

    // Estados de la aplicación

    const [page, setPage] = useState('home'); // 'home', 'tasks', 'rewards'

    const [userId, setUserId] = useState(null);

    const [appInitialized, setAppInitialized] = useState(false);

    const [points, setPoints] = useState(0);

    const [tasks, setTasks] = useState([]);

    const [db, setDb] = useState(null);

    const [auth, setAuth] = useState(null);

    const [notification, setNotification] = useState({ message: '', type: '' });

    const [loading, setLoading] = useState(false);



    // Definición de tareas EXCLUSIVAS de reciclaje y acción ecológica

    const BASE_TASKS = useMemo(() => [

        { id: 'recycle_plastic_bottles', text: 'Reciclar 10 botellas PET de plástico (limpias y aplastadas)', type: 'Reciclaje', points: 50, color: 'blue' },

        { id: 'recycle_carton_boxes', text: 'Reciclar una pila de cartón y cajas de embalaje', type: 'Reciclaje', points: 40, color: 'brown' },

        { id: 'eco_compost', text: 'Separar residuos orgánicos para compostaje', type: 'Acción Eco', points: 30, color: 'green' },

        { id: 'recycle_glass_jars', text: 'Reciclar 5 envases de vidrio (sin tapas)', type: 'Reciclaje', points: 60, color: 'cyan' },

        { id: 'eco_water_save', text: 'Reducir el tiempo de la ducha a menos de 5 minutos hoy', type: 'Acción Eco', points: 25, color: 'indigo' },

        { id: 'eco_battery_disposal', text: 'Llevar 5 pilas usadas a un punto de acopio oficial', type: 'Reciclaje', points: 75, color: 'yellow' },

        { id: 'eco_no_straw', text: 'Rechazar el uso de pitillos/pajitas de plástico hoy', type: 'Acción Eco', points: 15, color: 'emerald' },

    ], []);



    // Definición de recompensas temáticas (simulación)

    const REWARDS = useMemo(() => [

        { id: 'reward_pencil', name: 'Lápiz/Bolígrafo Ecológico', cost: 100, emoji: '✏️' },

        { id: 'reward_seed_kit', name: 'Kit de Semillas para plantar', cost: 250, emoji: '🌱' },

        { id: 'reward_bottle', name: 'Botella Reutilizable Personalizada', cost: 500, emoji: '💧' },

        { id: 'reward_shirt', name: 'Camiseta de Eco-Héroe del Colegio', cost: 1000, emoji: '👕' },

    ], []);



    // --- FIREBASE Y AUTENTICACIÓN ---



    const initializeFirebase = useCallback(async () => {

        if (!firebaseConfig) {

            setNotification({ message: 'Error: Configuración de Firebase no disponible.', type: 'error' });

            return;

        }

        try {

            const app = initializeApp(firebaseConfig);

            const firestore = getFirestore(app);

            const authInstance = getAuth(app);



            setDb(firestore);

            setAuth(authInstance);



            const unsubscribe = onAuthStateChanged(authInstance, async (currentUser) => {

                if (currentUser) {

                    setUserId(currentUser.uid);



                    const userRef = doc(firestore, `artifacts/${appId}/users/${currentUser.uid}/data/profile`);

                    const docSnap = await getDoc(userRef);



                    if (!docSnap.exists()) {

                        await setDoc(userRef, {

                            points: 0,

                            lastLogin: new Date().toISOString(),

                            currentTasks: BASE_TASKS.map(task => ({ ...task, completed: false }))

                        });

                        setTasks(BASE_TASKS.map(task => ({ ...task, completed: false })));

                        setPoints(0);

                    }

                    

                    setAppInitialized(true);

                } else {

                    if (initialAuthToken) {

                        await signInWithCustomToken(authInstance, initialAuthToken);

                    } else {

                        await signInAnonymously(authInstance);

                    }

                }

            });



            return unsubscribe; 



        } catch (e) {

            console.error("Error al inicializar o autenticar Firebase:", e);

            setNotification({ message: 'Error de conexión. Verifica tu red.', type: 'error' });

            setAppInitialized(true);

        }

    }, [BASE_TASKS]);



    useEffect(() => {

        initializeFirebase();

    }, [initializeFirebase]);



    // --- SUSCRIPCIÓN EN TIEMPO REAL ---

    useEffect(() => {

        if (db && userId) {

            const userRef = doc(db, `artifacts/${appId}/users/${userId}/data/profile`);

            

            const unsubscribe = onSnapshot(userRef, (docSnap) => {

                if (docSnap.exists()) {

                    const data = docSnap.data();

                    setPoints(data.points || 0);

                    setTasks(data.currentTasks || BASE_TASKS.map(task => ({ ...task, completed: false })));

                }

            }, (error) => {

                console.error("Error al escuchar cambios en Firestore:", error);

                setNotification({ message: 'Error al cargar datos en tiempo real.', type: 'error' });

            });



            return () => unsubscribe();

        }

    }, [db, userId, BASE_TASKS]);



    // --- MANEJO DE TAREAS Y PUNTOS ---



    const handleCompleteTask = async (taskId, taskPoints) => {

        if (!db || !userId) return;



        setLoading(true);

        const userRef = doc(db, `artifacts/${appId}/users/${userId}/data/profile`);



        try {

            const newTasks = tasks.map(t => 

                t.id === taskId ? { ...t, completed: true } : t

            );

            setTasks(newTasks);



            await updateDoc(userRef, {

                points: points + taskPoints,

                currentTasks: newTasks,

                completedTasks: arrayUnion({ taskId, date: new Date().toISOString() })

            });



            setNotification({ message: `¡Misión cumplida! Ganaste ${taskPoints} Eco-Puntos.`, type: 'success' });



        } catch (error) {

            console.error("Error al completar misión:", error);

            setNotification({ message: 'Error al guardar la misión. Intenta de nuevo.', type: 'error' });

        } finally {

            setLoading(false);

        }

    };



    const handleRedeemReward = async (reward) => {

        if (!db || !userId) return;

        if (points < reward.cost) {

            setNotification({ message: `Necesitas ${reward.cost - points} Eco-Puntos más. ¡A seguir reciclando!`, type: 'error' });

            return;

        }



        setLoading(true);

        const userRef = doc(db, `artifacts/${appId}/users/${userId}/data/profile`);



        try {

            await updateDoc(userRef, {

                points: points - reward.cost,

                redeemedRewards: arrayUnion({ rewardId: reward.id, name: reward.name, cost: reward.cost, date: new Date().toISOString() })

            });



            setNotification({ message: `¡Recompensa canjeada! Reclama tu ${reward.name}.`, type: 'success' });



        } catch (error) {

            console.error("Error al canjear recompensa:", error);

            setNotification({ message: 'Error al canjear. Intenta de nuevo.', type: 'error' });

        } finally {

            setLoading(false);

        }

    };



    // --- COMPONENTES DE VISTA ---



    const TaskCard = ({ task }) => (

        <div className={`p-4 mb-3 rounded-xl shadow-md flex justify-between items-center transition duration-300 ${

            task.completed 

                ? 'bg-gray-100 border-l-4 border-gray-400 opacity-60' 

                : 'bg-white hover:shadow-lg hover:border-l-4 border-l-4 border-white hover:border-l-emerald-500'

        }`}>

            <div>

                <span className={`font-semibold ${task.completed ? 'text-gray-500 line-through' : 'text-gray-700'}`}>

                    <TaskIcon type={task.type} /> {task.type}: {task.text}

                </span>

                <p className="text-sm text-amber-600 font-bold mt-1">+{task.points} Eco-Puntos</p>

            </div>

            <button

                onClick={() => handleCompleteTask(task.id, task.points)}

                disabled={task.completed || loading}

                className={`py-2 px-4 text-sm font-bold rounded-full transition duration-150 ${

                    task.completed 

                        ? 'bg-gray-300 text-gray-600 cursor-not-allowed' 

                        : 'bg-emerald-500 text-white hover:bg-emerald-600 shadow-md'

                }`}

            >

                {task.completed ? '✅ Logrado' : 'Marcar'}

            </button>

        </div>

    );

    

    // Función para íconos temáticos

    const TaskIcon = ({ type }) => {

        if (type === 'Reciclaje') return '♻️';

        if (type === 'Acción Eco') return '🌳';

        return '⭐';

    };





    const RewardCard = ({ reward }) => (

        <div className="bg-white p-4 rounded-xl shadow-lg flex flex-col justify-between items-center transition duration-300 hover:shadow-xl hover:scale-[1.02]">

            <span className="text-5xl mb-2">{reward.emoji}</span>

            <h3 className="text-lg font-bold text-gray-700 text-center mb-2">{reward.name}</h3>

            <p className="text-sm text-gray-500 mb-4">Incentivo por tu contribución.</p>

            <button

                onClick={() => handleRedeemReward(reward)}

                disabled={points < reward.cost || loading}

                className={`w-full py-2 text-md font-bold rounded-full transition duration-150 ${

                    points < reward.cost

                        ? 'bg-red-400 text-white opacity-70 cursor-not-allowed'

                        : 'bg-yellow-500 text-white hover:bg-yellow-600 shadow-md'

                }`}

            >

                {points < reward.cost 

                    ? `Faltan ${reward.cost - points}` 

                    : `Canjear por ${reward.cost} Puntos`}

            </button>

        </div>

    );



    const PageHome = () => (

        <div className="text-center p-6 bg-white rounded-xl shadow-lg">

            <h2 className="text-3xl font-extrabold text-emerald-600 mb-4">¡Bienvenido, Eco-Héroe!</h2>

            <p className="text-xl text-gray-600 mb-6">Tu misión es simple: Reciclar, Cuidar, Ganar.</p>

            <div className="bg-lime-50 p-4 rounded-lg border-2 border-lime-300 shadow-inner">

                <p className="text-5xl font-extrabold text-lime-700">{points}</p>

                <p className="text-lg font-semibold text-lime-800">Eco-Puntos Acumulados</p>

            </div>



            <div className="mt-8 space-y-4">

                <button

                    onClick={() => setPage('tasks')}

                    className="w-full py-3 bg-emerald-600 text-white font-bold rounded-xl text-lg shadow-md hover:bg-emerald-700 transition duration-150 flex items-center justify-center"

                >

                    <span className="mr-2">🌳 Misiones Ecológicas</span>

                    <svg xmlns="http://www.w3.org/2000/svg" className="h-5 w-5" viewBox="0 0 20 20" fill="currentColor">

                        <path fillRule="evenodd" d="M4 2a2 2 0 00-2 2v12a2 2 0 002 2h12a2 2 0 002-2V4a2 2 0 00-2-2H4zm-1 9a1 1 0 011-1h12a1 1 0 110 2H4a1 1 0 01-1-1z" clipRule="evenodd" />

                    </svg>

                </button>

                <button

                    onClick={() => setPage('rewards')}

                    className="w-full py-3 bg-indigo-500 text-white font-bold rounded-xl text-lg shadow-md hover:bg-indigo-600 transition duration-150 flex items-center justify-center"

                >

                    <span className="mr-2">🎁 Tienda de Eco-Premios</span>

                    <svg xmlns="http://www.w3.org/2000/svg" className="h-5 w-5" viewBox="0 0 20 20" fill="currentColor">

                        <path d="M5.5 12.5l5 5 5-5m-10-5l5-5 5 5" />

                    </svg>

                </button>

            </div>

        </div>

    );



    const PageTasks = () => (

        <div className="p-6 bg-white rounded-xl shadow-lg">

            <h2 className="text-3xl font-bold text-gray-800 mb-6">🌳 Misiones Ecológicas Diarias</h2>

            <p className="text-sm text-gray-500 mb-4">Cada acción suma puntos y mejora el planeta.</p>

            <div className="space-y-4">

                {tasks.length > 0 ? (

                    tasks.map(task => <TaskCard key={task.id} task={task} />)

                ) : (

                    <p className="text-center text-gray-400">Cargando misiones...</p>

                )}

            </div>

        </div>

    );



    const PageRewards = () => (

        <div className="p-6 bg-white rounded-xl shadow-lg">

            <h2 className="text-3xl font-bold text-gray-800 mb-6 flex justify-between items-center">

                <span>🎁 Tienda de Eco-Premios</span>

                <span className="text-xl font-extrabold text-lime-700 bg-lime-100 px-3 py-1 rounded-full">{points} Puntos</span>

            </h2>

            <p className="text-sm text-gray-500 mb-6">Canjea tus Eco-Puntos por recompensas que te ayudarán a seguir cuidando el planeta.</p>

            <div className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-4 gap-6">

                {REWARDS.map(reward => <RewardCard key={reward.id} reward={reward} />)}

            </div>

        </div>

    );



    // --- RENDERIZADO PRINCIPAL ---



    if (!appInitialized) {

        return (

            <div className="flex items-center justify-center min-h-screen bg-emerald-50">

                <div className="text-center p-8 rounded-xl bg-white shadow-2xl">

                    <div className="animate-spin rounded-full h-12 w-12 border-b-2 border-emerald-500 mx-auto mb-4"></div>

                    <p className="text-lg font-semibold text-gray-700">Conectando con la Eco-Red...</p>

                    <p className="text-sm text-gray-500 mt-2">Iniciando sesión de Eco-Héroe.</p>

                </div>

            </div>

        );

    }



    const currentPage = {

        'home': <PageHome />,

        'tasks': <PageTasks />,

        'rewards': <PageRewards />,

    }[page];



    return (

        <div className="min-h-screen bg-emerald-50 p-4 sm:p-8">

            {/* Notificación Flotante */}

            {notification.message && (

                <div 

                    className={`fixed top-4 left-1/2 transform -translate-x-1/2 px-6 py-3 rounded-full shadow-lg text-white font-semibold transition-all duration-300 z-50 ${

                        notification.type === 'success' ? 'bg-green-600' : 'bg-red-600'

                    }`}

                    onClick={() => setNotification({ message: '', type: '' })}

                >

                    {notification.message}

                </div>

            )}

            

            <header className="text-center mb-8">

                <h1 className="text-4xl font-bold text-gray-800">

                    ECO-HÉROE 🌳

                </h1>

                <p className="text-sm text-gray-500">ID de Héroe: <span className="font-mono text-xs bg-gray-200 p-1 rounded-sm">{userId || 'Cargando...'}</span></p>

            </header>



            <main className="max-w-4xl mx-auto pb-12">

                {currentPage}

            </main>



            {/* Barra de Navegación Fija (Responsive) */}

            <div className="fixed bottom-0 left-0 right-0 bg-white border-t border-gray-200 shadow-xl">

                <nav className="flex justify-around max-w-lg mx-auto py-2">

                    <NavButton pageKey="home" currentPage={page} setPage={setPage} icon="🏠" label="Inicio" />

                    <NavButton pageKey="tasks" currentPage={page} setPage={setPage} icon="🌳" label="Misiones" />

                    <NavButton pageKey="rewards" currentPage={page} setPage={setPage} icon="🎁" label="Premios" />

                </nav>

            </div>

        </div>

    );

};



// Componente para botones de navegación

const NavButton = ({ pageKey, currentPage, setPage, icon, label }) => (

    <button

        onClick={() => setPage(pageKey)}

        className={`flex flex-col items-center p-2 rounded-lg transition duration-150 ease-in-out ${

            currentPage === pageKey 

                ? 'text-emerald-600 bg-emerald-100 shadow-inner' 

                : 'text-gray-500 hover:text-emerald-500'

        }`}

    >

        <span className="text-2xl">{icon}</span>

        <span className="text-xs font-semibold mt-1 hidden sm:block">{label}</span>

    </button>

);



export default App;
