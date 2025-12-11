<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>智能衣櫃管理系統</title>
    <!-- 載入 Tailwind CSS -->
    <script src="https://cdn.tailwindcss.com"></script>
    <!-- 載入 Lucide Icons -->
    <script src="https://unpkg.com/lucide@latest"></script>
    <style>
        /* 確保字體清晰度 */
        body { font-family: 'Inter', sans-serif; }
        /* 自定義滾動條樣式 */
        .scroll-container::-webkit-scrollbar { width: 8px; }
        .scroll-container::-webkit-scrollbar-thumb { background-color: #a78bfa; border-radius: 4px; }
        .scroll-container::-webkit-scrollbar-track { background: #f3f4f6; }
    </style>
</head>
<body class="bg-gray-50 min-h-screen">

    <div id="app" class="p-4 sm:p-6">
        <!-- React component will render here -->
        <div class="flex items-center justify-center min-h-[50vh] text-indigo-600">
            <svg class="animate-spin -ml-1 mr-3 h-5 w-5 text-indigo-600" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
                <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle>
                <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
            </svg>
            載入應用程式...
        </div>
    </div>

    <!-- 載入 React and ReactDOM -->
    <script crossorigin src="https://unpkg.com/react@18/umd/react.development.js"></script>
    <script crossorigin src="https://unpkg.com/react-dom@18/umd/react-dom.development.js"></script>
    <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
    
    <!-- 載入 Firebase 模組 -->
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { 
            getFirestore, collection, onSnapshot, addDoc, deleteDoc, doc, query, orderBy, serverTimestamp, getDocs
        } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        // 全域變數供 React 程式碼使用
        window.firebaseDependencies = {
            initializeApp, getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged,
            getFirestore, collection, onSnapshot, addDoc, deleteDoc, doc, query, orderBy, serverTimestamp, getDocs
        };

        // 從環境變數中取得配置
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
        // 如果 __firebase_config 未定義，將其設為空物件 {}，但會被後續的 React 邏輯攔截並報錯。
        //const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {};
		const firebaseConfig = {
			apiKey: "AIzaSyBOdM7pZVxknhHve33b8m4HmBs4hgmLbgQ",
			authDomain: "closet-482eb.firebaseapp.com",
			projectId: "closet-482eb",
			storageBucket: "closet-482eb.firebasestorage.app",
			messagingSenderId: "584513538498",
			appId: "1:584513538498:web:2f900e1d69cf02aef62e5b",
			measurementId: "G-EDV42FT5VE"
		  };
        const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;

        window.firebaseConfigData = { appId, firebaseConfig, initialAuthToken };
    </script>
    
    <script type="text/babel">
        const { useState, useEffect, useMemo } = React;
        const { initializeApp, getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged,
                getFirestore, collection, onSnapshot, addDoc, deleteDoc, doc, query, orderBy, serverTimestamp } = window.firebaseDependencies || {};
        const { appId, firebaseConfig, initialAuthToken } = window.firebaseConfigData || {};

        // 使用 Lucide 圖標的簡化組件
        const Icon = ({ name, size = 24, className = 'text-gray-700' }) => {
            const iconData = lucide.icons[name];
            if (!iconData) return <span className={className}>?</span>;
            return (
                <svg xmlns="http://www.w3.org/2000/svg" width={size} height={size} viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round" className={className}>
                    {iconData.map((path, index) => <path key={index} d={path} />)}
                </svg>
            );
        };

        // 常數定義
        const CATEGORIES = ['上衣', '下裝', '外套', '鞋子', '配飾'];
        const COLORS = ['白色', '黑色', '灰色', '藍色', '紅色', '綠色', '黃色', '其他'];

        // Helper function to get the collection path for private user data
        const getCollectionPath = (uid, collectionName) => `/artifacts/${appId}/users/${uid}/${collectionName}`;

        const getColorCode = (colorName) => {
            switch (colorName) {
                case '白色': return '#f3f4f6';
                case '黑色': return '#1f2937';
                case '灰色': return '#9ca3af';
                case '藍色': return '#3b82f6';
                case '紅色': return '#ef4444';
                case '綠色': return '#10b981';
                case '黃色': return '#f59e0b';
                default: return '#6b7280';
            }
        };

        // --- Core Application Component ---
        const App = () => {
            const [db, setDb] = useState(null);
            const [userId, setUserId] = useState(null);
            const [isLoading, setIsLoading] = useState(true);
            const [error, setError] = useState(null);

            const [items, setItems] = useState([]);
            const [outfits, setOutfits] = useState([]);
            const [activeTab, setActiveTab] = useState('wardrobe'); 

            // State for adding new item, imageSource stores Base64 or URL
            const [newItem, setNewItem] = useState({ name: '', category: CATEGORIES[0], color: COLORS[0], imageSource: '' });
            
            // State for outfit planner
            const [selectedItems, setSelectedItems] = useState({});
            const [newOutfitName, setNewOutfitName] = useState('');

            // --- Firebase Initialization and Authentication ---
            useEffect(() => {
                const initializeFirebase = async () => {
                    // 1. 檢查 Firebase 依賴項是否載入
                    if (!initializeApp || !firebaseConfig || !getAuth || !getFirestore) {
                        setError("Firebase 依賴項載入失敗。");
                        setIsLoading(false);
                        return;
                    }
                    
                    // 2. 新增檢查：如果 firebaseConfig 為空物件，則假設在本地環境運行且配置遺失。
                    if (Object.keys(firebaseConfig).length === 0) {
                        console.error("Firebase config is empty. Initialization aborted.");
                        setError("Firebase配置不完整或遺失。如果您下載後在本地運行，這表示應用程式無法連接到數據庫。請確認您是在 Canvas 環境中運行，或已手動提供有效的 Firebase Config。");
                        setIsLoading(false);
                        return;
                    }

                    try {
                        const app = initializeApp(firebaseConfig);
                        const authInstance = getAuth(app);
                        const dbInstance = getFirestore(app);
                        setDb(dbInstance);

                        const unsubscribe = onAuthStateChanged(authInstance, async (user) => {
                            if (!user) {
                                try {
                                    if (initialAuthToken) {
                                        await signInWithCustomToken(authInstance, initialAuthToken);
                                    } else {
                                        await signInAnonymously(authInstance);
                                    }
                                } catch (e) {
                                    console.error("Firebase Sign-in Failed:", e);
                                    setError("身份驗證失敗。");
                                    setIsLoading(false);
                                }
                            } else {
                                setUserId(user.uid);
                                setIsLoading(false);
                            }
                        });
                        return () => unsubscribe();
                    } catch (e) {
                        console.error("Firebase Initialization Error:", e);
                        setError("Firebase初始化失敗。");
                        setIsLoading(false);
                    }
                };
                initializeFirebase();
            }, []);

            // --- Firestore Data Fetching (Real-time Listeners) ---
            useEffect(() => {
                if (db && userId) {
                    const setupListeners = (collectionName, setState) => {
                        const colRef = collection(db, getCollectionPath(userId, collectionName));
                        const q = query(colRef, orderBy('createdAt', 'desc'));
                        
                        return onSnapshot(q, (snapshot) => {
                            const fetchedData = snapshot.docs.map(doc => ({
                                id: doc.id,
                                ...doc.data()
                            }));
                            setState(fetchedData);
                        }, (e) => {
                            console.error(`Error fetching ${collectionName}:`, e);
                            setError(`無法載入 ${collectionName} 資料。`);
                        });
                    };

                    const unsubscribeItems = setupListeners('wardrobe_items', setItems);
                    const unsubscribeOutfits = setupListeners('saved_outfits', setOutfits);
                    
                    return () => {
                        unsubscribeItems();
                        unsubscribeOutfits();
                    };
                }
            }, [db, userId]);

            // --- CRUD Operations ---
            const handleAddItem = async (e) => {
                e.preventDefault();
                if (!db || !userId || !newItem.name.trim()) return;

                try {
                    await addDoc(collection(db, getCollectionPath(userId, 'wardrobe_items')), {
                        ...newItem,
                        name: newItem.name.trim(),
                        imageSource: newItem.imageSource.trim(),
                        createdAt: serverTimestamp(),
                    });
                    setNewItem(prev => ({ ...prev, name: '', imageSource: '' }));
                } catch (e) {
                    // Check for size error (Document too large)
                    if (e.message.includes("larger than 1,048,576 bytes")) {
                         setError("儲存失敗：圖片檔案太大，請使用較小的圖片（Base64 限制）。");
                    } else {
                        console.error("Error adding document: ", e);
                        setError("新增衣物失敗。");
                    }
                }
            };

            const handleDeleteItem = async (itemId) => {
                if (!db || !userId) return;
                try {
                    await deleteDoc(doc(db, getCollectionPath(userId, 'wardrobe_items'), itemId));
                    setSelectedItems(prev => {
                        const newSelected = { ...prev };
                        CATEGORIES.forEach(cat => {
                            if (newSelected[cat] && newSelected[cat].id === itemId) delete newSelected[cat];
                        });
                        return newSelected;
                    });
                } catch (e) {
                    console.error("Error deleting document: ", e);
                    setError("刪除衣物失敗。");
                }
            };

            // Handler for reading image file and converting to Base64
            const handleImageUpload = (event) => {
                const file = event.target.files[0];
                if (!file) return;

                // Warning for large file size (Firestore limit is 1MB)
                if (file.size > 500 * 1024) { 
                    console.warn("Warning: Image size is large. Max recommended size for Firestore is <500KB.");
                    // You might want to display a user-friendly warning here
                }

                const reader = new FileReader();
                reader.onloadend = () => {
                    setNewItem(prev => ({ ...prev, imageSource: reader.result }));
                };
                reader.onerror = (error) => {
                    console.error("Error reading file:", error);
                    setError("無法讀取圖片檔案。");
                };
                reader.readAsDataURL(file);
            };

            // --- UI Components ---
            
            const ItemCard = ({ item, onDelete, isSelectable, isSelected, onSelect, size = 'lg' }) => {
                const [imgError, setImgError] = useState(false);
                const hasImage = item.imageSource && !imgError;
                
                useEffect(() => { setImgError(false); }, [item.id]);

                const cardClasses = size === 'lg' ? 'w-16 h-16 rounded-full' : 'w-4 h-4 rounded-full';
                const iconSize = size === 'lg' ? 32 : 14;

                return (
                    <div 
                        className={`relative p-3 bg-white rounded-lg shadow-md flex flex-col items-center transition-all duration-200 
                            ${isSelectable ? 'cursor-pointer hover:shadow-lg' : ''}
                            ${isSelected ? 'border-4 border-indigo-500 ring-2 ring-indigo-500' : 'border border-gray-100'}
                        `}
                        onClick={isSelectable ? () => onSelect(item) : undefined}
                    >
                        {/* Image / Placeholder Area */}
                        <div 
                            className={`${cardClasses} overflow-hidden flex items-center justify-center mb-2 text-xl font-bold text-white bg-gray-200`}
                            style={{ 
                                backgroundColor: hasImage ? 'transparent' : getColorCode(item.color),
                                border: hasImage ? 'none' : '1px solid #e5e7eb'
                            }}
                        >
                            {hasImage ? (
                                <img
                                    src={item.imageSource}
                                    alt={item.name}
                                    className="w-full h-full object-cover"
                                    onError={() => setImgError(true)}
                                />
                            ) : (
                                <div className='flex flex-col items-center justify-center text-gray-700 h-full w-full'>
                                    {imgError ? <Icon name="ImageOff" size={iconSize} /> : <Icon name="Shirt" size={iconSize} />}
                                </div>
                            )}
                        </div>

                        {size === 'lg' && (
                            <>
                                <p className="text-sm font-semibold text-gray-800 truncate w-full text-center">{item.name}</p>
                                <div className="text-xs text-gray-500 mt-1 flex space-x-2">
                                    <span>{item.category}</span>
                                    <span className="flex items-center">
                                        <Icon name="Palette" size={10} className="mr-0.5" /> {item.color}
                                    </span>
                                </div>
                            </>
                        )}
                        
                        {onDelete && (
                            <button
                                onClick={(e) => { e.stopPropagation(); onDelete(item.id); }}
                                className="absolute top-1 right-1 p-1 bg-red-100 text-red-600 rounded-full hover:bg-red-200 transition-colors"
                                aria-label="刪除衣物"
                            >
                                <Icon name="Trash2" size={14} />
                            </button>
                        )}

                        {isSelected && (
                            <div className="absolute top-0 left-0 bg-indigo-500 p-1 rounded-tl-lg rounded-br-lg text-white">
                                <Icon name="Check" size={14} />
                            </div>
                        )}
                    </div>
                );
            };

            const TabButton = ({ label, iconName, active, onClick }) => (
                <button
                    onClick={onClick}
                    className={`flex-1 flex items-center justify-center py-2 px-4 rounded-lg text-sm font-semibold transition-all duration-300
                    ${active 
                        ? 'bg-indigo-600 text-white shadow-lg' 
                        : 'text-gray-600 hover:bg-gray-100 hover:text-indigo-600'
                    }`
                    }
                >
                    <Icon name={iconName} size={20} className="mr-2 hidden sm:inline-block" />
                    {label}
                </button>
            );

            // --- Tab: Wardrobe Management ---
            const WardrobeSection = () => (
                <div className="space-y-6 p-4 md:p-6 bg-white rounded-xl shadow-lg">
                    <h2 className="text-2xl font-bold text-gray-800 border-b pb-2 mb-4">新增衣物</h2>
                    
                    <form onSubmit={handleAddItem} className="grid grid-cols-1 md:grid-cols-3 gap-4 bg-gray-50 p-4 rounded-lg shadow-inner">
                        
                        <div>
                            <label htmlFor="itemName" className="block text-sm font-medium text-gray-700">名稱 <span className="text-red-500">*</span></label>
                            <input
                                id="itemName" type="text" required value={newItem.name}
                                onChange={(e) => setNewItem(prev => ({ ...prev, name: e.target.value }))}
                                placeholder="例如：藍色牛仔褲"
                                className="mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-indigo-500 focus:ring-indigo-500 p-2"
                            />
                        </div>

                        <div>
                            <label htmlFor="itemCategory" className="block text-sm font-medium text-gray-700">類別</label>
                            <select
                                id="itemCategory" value={newItem.category}
                                onChange={(e) => setNewItem(prev => ({ ...prev, category: e.target.value }))}
                                className="mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-indigo-500 focus:ring-indigo-500 p-2"
                            >
                                {CATEGORIES.map(cat => <option key={cat} value={cat}>{cat}</option>)}
                            </select>
                        </div>

                        <div>
                            <label htmlFor="itemColor" className="block text-sm font-medium text-gray-700">顏色</label>
                            <select
                                id="itemColor" value={newItem.color}
                                onChange={(e) => setNewItem(prev => ({ ...prev, color: e.target.value }))}
                                className="mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-indigo-500 focus:ring-indigo-500 p-2"
                            >
                                {COLORS.map(color => <option key={color} value={color}>{color}</option>)}
                            </select>
                        </div>
                        
                        <div className="md:col-span-3">
                            <label htmlFor="imageFile" className="block text-sm font-medium text-gray-700">選擇圖片檔案 (手機/電腦)</label>
                            <input
                                id="imageFile" type="file" accept="image/*"
                                onChange={handleImageUpload}
                                className="mt-1 block w-full text-sm text-gray-500 file:mr-4 file:py-2 file:px-4 file:rounded-full file:border-0 file:text-sm file:font-semibold file:bg-indigo-50 file:text-indigo-700 hover:file:bg-indigo-100"
                            />
                            <p className="text-xs text-red-600 mt-1">⚠️ **警告：圖片將以 Base64 形式儲存，請確保檔案小於 500KB，避免資料庫儲存失敗。**</p>
                        </div>
                        
                        <div className="md:col-span-3 flex justify-end">
                            <button
                                type="submit"
                                className="px-4 py-2 bg-indigo-600 text-white font-semibold rounded-md shadow-lg hover:bg-indigo-700 transition-colors flex items-center"
                                disabled={!newItem.name.trim()}
                            >
                                <Icon name="Plus" size={18} className="mr-2" /> 新增至衣櫃
                            </button>
                        </div>
                    </form>

                    <h2 className="text-2xl font-bold text-gray-800 border-b pb-2 mb-4 mt-8">我的衣物 ({items.length} 件)</h2>
                    {items.length === 0 ? (
                        <p className="text-gray-500 text-center py-10 bg-gray-50 rounded-lg shadow-md">衣櫃是空的！請新增一些衣物。</p>
                    ) : (
                        <div className="grid grid-cols-2 sm:grid-cols-3 md:grid-cols-4 lg:grid-cols-5 xl:grid-cols-6 gap-4">
                            {items.map(item => (
                                <ItemCard key={item.id} item={item} onDelete={handleDeleteItem} size="lg" />
                            ))}
                        </div>
                    )}
                </div>
            );

            // --- Simplified Planner and Suggestions Sections (for brevity and focus on core change) ---
            const PlannerSection = () => {
                const isItemSelected = (item) => selectedItems[item.category]?.id === item.id;

                const handleItemSelect = (item) => {
                    setSelectedItems(prev => {
                        const category = item.category;
                        if (prev[category]?.id === item.id) {
                            const newState = { ...prev };
                            delete newState[category];
                            return newState;
                        }
                        return { ...prev, [category]: item };
                    });
                };
                
                const handleSaveOutfit = async (e) => {
                    e.preventDefault();
                    if (!db || !userId || !newOutfitName.trim() || Object.keys(selectedItems).length === 0) return;
                
                    try {
                        await addDoc(collection(db, getCollectionPath(userId, 'saved_outfits')), {
                            name: newOutfitName.trim(),
                            items: Object.values(selectedItems),
                            createdAt: serverTimestamp(),
                        });
                        setNewOutfitName('');
                        setSelectedItems({});
                    } catch (e) {
                        console.error("Error saving outfit: ", e);
                        setError("儲存搭配失敗。");
                    }
                };
                
                const handleDeleteOutfit = async (outfitId) => {
                    if (!db || !userId) return;
                    try {
                        await deleteDoc(doc(db, getCollectionPath(userId, 'saved_outfits'), outfitId));
                    } catch (e) {
                        console.error("Error deleting outfit: ", e);
                        setError("刪除搭配失敗。");
                    }
                };


                return (
                    <div className="space-y-6 p-4 md:p-6 bg-white rounded-xl shadow-lg">
                        <h2 className="text-2xl font-bold text-gray-800 border-b pb-2 mb-4">搭配規劃</h2>
                        
                        <div className="flex flex-wrap gap-4 justify-center p-4 bg-gray-50 rounded-lg shadow-inner">
                            {CATEGORIES.map(cat => (
                                <div key={cat} className="w-full sm:w-1/2 md:w-1/4 lg:w-1/6 p-2 text-center">
                                    <p className="text-sm font-semibold text-gray-600 mb-1">{cat}</p>
                                    {selectedItems[cat] ? (
                                        <div className="relative border border-green-300 bg-green-50 p-2 rounded-lg shadow-inner">
                                            <ItemCard item={selectedItems[cat]} size="sm" />
                                            <p className="text-xs mt-1 font-medium">{selectedItems[cat].name}</p>
                                            <button 
                                                onClick={() => handleItemSelect(selectedItems[cat])}
                                                className="absolute top-0 right-0 p-1 text-xs bg-red-500 text-white rounded-bl-lg"
                                            >
                                                X
                                            </button>
                                        </div>
                                    ) : (
                                        <div className="border border-dashed border-gray-300 p-4 h-24 flex items-center justify-center text-gray-400 rounded-lg">
                                            選擇 {cat}
                                        </div>
                                    )}
                                </div>
                            ))}
                        </div>
                        
                        <form onSubmit={handleSaveOutfit} className="flex flex-col sm:flex-row gap-3 mt-4">
                            <input
                                type="text"
                                value={newOutfitName}
                                onChange={(e) => setNewOutfitName(e.target.value)}
                                placeholder="為這套搭配命名"
                                className="flex-grow p-2 rounded-md border-gray-300 shadow-sm focus:border-indigo-500 focus:ring-indigo-500"
                                required
                            />
                            <button
                                type="submit"
                                className="px-4 py-2 bg-pink-500 text-white font-semibold rounded-md shadow-lg hover:bg-pink-600 transition-colors flex items-center justify-center"
                                disabled={!newOutfitName.trim() || Object.keys(selectedItems).length === 0}
                            >
                                <Icon name="Zap" size={18} className="mr-2" /> 儲存搭配
                            </button>
                        </form>

                        <h3 className="text-xl font-bold text-gray-800 border-b pb-2 mb-4 mt-8">選擇單品</h3>
                        <div className="scroll-container overflow-y-auto max-h-[40vh] p-2">
                            {items.length === 0 ? (
                                <p className="text-gray-500 text-center py-10 bg-white rounded-lg shadow-md">請先新增衣物。</p>
                            ) : (
                                CATEGORIES.map(category => (
                                    <div key={category} className="mb-6">
                                        <h4 className="text-lg font-semibold text-gray-700 mb-3 p-2 bg-indigo-100 rounded-md">{category}</h4>
                                        <div className="grid grid-cols-3 md:grid-cols-4 lg:grid-cols-6 gap-4">
                                            {items.filter(item => item.category === category).map(item => (
                                                <ItemCard
                                                    key={item.id} item={item} isSelectable
                                                    isSelected={isItemSelected(item)} onSelect={handleItemSelect}
                                                />
                                            ))}
                                        </div>
                                    </div>
                                ))
                            )}
                        </div>
                        
                        <h3 className="text-xl font-bold text-gray-800 border-b pb-2 mb-4 mt-8">已儲存的搭配 ({outfits.length})</h3>
                        <div className="space-y-4">
                            {outfits.length === 0 ? (
                                <p className="text-gray-500 text-center bg-gray-50 p-4 rounded-lg shadow-md">您還沒有儲存任何搭配。</p>
                            ) : (
                                outfits.map(outfit => (
                                    <div key={outfit.id} className="bg-gray-50 p-4 rounded-lg shadow-md border border-gray-100 relative">
                                        <div className="flex justify-between items-center mb-3">
                                            <h4 className="text-lg font-bold text-indigo-600">{outfit.name}</h4>
                                            <button
                                                onClick={() => handleDeleteOutfit(outfit.id)}
                                                className="p-1 bg-red-100 text-red-600 rounded-full hover:bg-red-200 transition-colors"
                                                aria-label="刪除搭配"
                                            >
                                                <Icon name="Trash2" size={18} />
                                            </button>
                                        </div>
                                        <div className="flex flex-wrap gap-3">
                                            {outfit.items.map(item => (
                                                <div key={item.id} className="flex items-center space-x-2 bg-white p-2 rounded-full text-sm">
                                                    <div className="w-4 h-4 rounded-full overflow-hidden" style={{ backgroundColor: item.imageSource ? 'transparent' : getColorCode(item.color) }}>
                                                        {item.imageSource && <img src={item.imageSource} alt={item.name} className="w-full h-full object-cover" onError={(e) => { e.target.style.display = 'none'; }}/>}
                                                    </div>
                                                    <span>{item.name}</span>
                                                </div>
                                            ))}
                                        </div>
                                    </div>
                                ))
                            )}
                        </div>
                    </div>
                );
            };

            const SuggestionsSection = () => {
                const [simulatedWeather, setSimulatedWeather] = useState('涼爽');
                
                const suggestedOutfit = useMemo(() => {
                    const suggestions = {};
                    if (simulatedWeather === '溫暖') {
                        suggestions['上衣'] = items.filter(i => i.category === '上衣' && i.color !== '黑色').sort(() => 0.5 - Math.random())[0];
                        suggestions['下裝'] = items.filter(i => i.category === '下裝' && i.color === '藍色').sort(() => 0.5 - Math.random())[0];
                    } else if (simulatedWeather === '涼爽') {
                        suggestions['外套'] = items.filter(i => i.category === '外套').sort(() => 0.5 - Math.random())[0];
                        suggestions['上衣'] = items.filter(i => i.category === '上衣' && i.color === '黑色').sort(() => 0.5 - Math.random())[0];
                        suggestions['下裝'] = items.filter(i => i.category === '下裝').sort(() => 0.5 - Math.random())[0];
                    }
                    return Object.values(suggestions).filter(Boolean);
                }, [items, simulatedWeather]);
            
                return (
                    <div className="space-y-6 p-4 md:p-6 bg-white rounded-xl shadow-lg">
                        <h2 className="text-2xl font-bold text-gray-800 border-b pb-2 mb-4">穿搭建議 (模擬天氣)</h2>
                        
                        <div className="bg-gray-50 p-4 rounded-lg shadow-inner flex items-center justify-between">
                            <label className="text-lg font-medium text-gray-700 flex items-center">
                                <Icon name="Sun" size={24} className="text-yellow-500 mr-2" /> 模擬當前天氣：
                            </label>
                            <div className="flex space-x-4">
                                <button
                                    onClick={() => setSimulatedWeather('溫暖')}
                                    className={`py-2 px-4 rounded-full font-semibold transition-all ${simulatedWeather === '溫暖' ? 'bg-orange-500 text-white shadow-md' : 'bg-gray-200 text-gray-700 hover:bg-orange-100'}`}
                                >
                                    ☀️ 溫暖
                                </button>
                                <button
                                    onClick={() => setSimulatedWeather('涼爽')}
                                    className={`py-2 px-4 rounded-full font-semibold transition-all ${simulatedWeather === '涼爽' ? 'bg-blue-500 text-white shadow-md' : 'bg-gray-200 text-gray-700 hover:bg-blue-100'}`}
                                >
                                    ☁️ 涼爽
                                </button>
                            </div>
                        </div>

                        <h3 className="text-xl font-bold text-gray-800 border-b pb-2 mb-4 mt-8">今日推薦搭配 ({simulatedWeather})</h3>
                        {suggestedOutfit.length > 0 ? (
                            <div className="grid grid-cols-2 sm:grid-cols-3 md:grid-cols-4 lg:grid-cols-5 gap-4">
                                {suggestedOutfit.map(item => (
                                    <div key={item.id} className="p-3 bg-white rounded-lg shadow-lg flex flex-col items-center border-2 border-indigo-400">
                                        <div 
                                            className="w-16 h-16 rounded-full overflow-hidden flex items-center justify-center mb-2 text-xl font-bold text-white"
                                            style={{ backgroundColor: item.imageSource ? 'transparent' : getColorCode(item.color) }}
                                        >
                                            {item.imageSource ? (
                                                <img src={item.imageSource} alt={item.name} className="w-full h-full object-cover" onError={(e) => { e.target.style.display = 'none'; e.target.parentNode.innerHTML = '<svg xmlns="http://www.w3.org/2000/svg" width="32" height="32" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round" class="lucide lucide-zap"><path d="M10.82 1L8 10h8l-2.82 9" /><path d="M12 21l-2.5-5" /><path d="M12 3l-2.5 5" /></svg>'; }}/>
                                            ) : (
                                                <Icon name="Zap" size={32} />
                                            )}
                                        </div>
                                        <p className="text-sm font-bold text-gray-800">{item.name}</p>
                                        <p className="text-xs text-indigo-600 font-semibold mt-1">{item.category}</p>
                                    </div>
                                ))}
                            </div>
                        ) : (
                            <p className="text-gray-500 text-center py-10 bg-gray-50 rounded-lg shadow-md">您的衣櫃中沒有足夠的衣物來進行此類建議。</p>
                        )}
                    </div>
                );
            };

            // --- Main Render ---

            if (isLoading) {
                return (
                    <div className="flex items-center justify-center min-h-[50vh] text-indigo-600">
                        <svg className="animate-spin -ml-1 mr-3 h-5 w-5 text-indigo-600" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
                            <circle className="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" strokeWidth="4"></circle>
                            <path className="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
                        </svg>
                        載入中... (正在進行身份驗證並連接到 Firestore)
                    </div>
                );
            }

            if (error) {
                return (
                    <div className="max-w-4xl mx-auto p-8 text-center text-red-600 bg-red-100 border border-red-300 rounded-lg shadow-lg mt-10">
                        <p className="font-bold mb-2">發生錯誤：</p>
                        <p>{error}</p>
                        <p className="mt-4 text-sm text-gray-700">用戶ID: {userId || 'N/A'}</p>
                    </div>
                );
            }

            return (
                <div className="max-w-4xl mx-auto">
                    <header className="text-center mb-6">
                        <h1 className="text-4xl font-extrabold text-indigo-700 flex items-center justify-center">
                            <Icon name="Shirt" size={36} className="mr-3" />
                            智能衣櫃管理系統
                        </h1>
                        <p className="text-sm text-gray-500 mt-2">
                            用戶 ID: {userId} (資料儲存於 Firebase)
                        </p>
                    </header>

                    {/* Tab Navigation */}
                    <nav className="mb-6 flex space-x-1 bg-white p-2 rounded-xl shadow-lg">
                        <TabButton label="我的衣櫃" iconName="Tally3" active={activeTab === 'wardrobe'} onClick={() => setActiveTab('wardrobe')} />
                        <TabButton label="搭配規劃" iconName="Palette" active={activeTab === 'planner'} onClick={() => setActiveTab('planner')} />
                        <TabButton label="穿搭建議" iconName="Zap" active={activeTab === 'suggestions'} onClick={() => setActiveTab('suggestions')} />
                    </nav>

                    <main>
                        {activeTab === 'wardrobe' && <WardrobeSection />}
                        {activeTab === 'planner' && <PlannerSection />}
                        {activeTab === 'suggestions' && <SuggestionsSection />}
                    </main>

                    <footer className="mt-8 text-center text-sm text-gray-400">
                        <p>© 智能衣櫃 (HTML/React 單一文件版本)</p>
                    </footer>
                </div>
            );
        };

        // 渲染應用程式
        ReactDOM.render(<App />, document.getElementById('app'));

    </script>
	
	<!--
	<script type="module">
	  // Import the functions you need from the SDKs you need
	  import { initializeApp } from "https://www.gstatic.com/firebasejs/12.6.0/firebase-app.js";
	  import { getAnalytics } from "https://www.gstatic.com/firebasejs/12.6.0/firebase-analytics.js";
	  // TODO: Add SDKs for Firebase products that you want to use
	  // https://firebase.google.com/docs/web/setup#available-libraries

	  // Your web app's Firebase configuration
	  // For Firebase JS SDK v7.20.0 and later, measurementId is optional
	  const firebaseConfig = {
		apiKey: "AIzaSyBOdM7pZVxknhHve33b8m4HmBs4hgmLbgQ",
		authDomain: "closet-482eb.firebaseapp.com",
		projectId: "closet-482eb",
		storageBucket: "closet-482eb.firebasestorage.app",
		messagingSenderId: "584513538498",
		appId: "1:584513538498:web:2f900e1d69cf02aef62e5b",
		measurementId: "G-EDV42FT5VE"
	  };

	  // Initialize Firebase
	  const app = initializeApp(firebaseConfig);
	  const analytics = getAnalytics(app);
	</script>
	-->
</body>
</html>
