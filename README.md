<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>單字聽力扭蛋機</title>
    <!-- 引入 Tailwind CSS -->
    <script src="https://cdn.tailwindcss.com"></script>
    <!-- 引入 React 和 ReactDOM -->
    <script src="https://unpkg.com/react@18/umd/react.production.min.js"></script>
    <script src="https://unpkg.com/react-dom@18/umd/react-dom.production.min.js"></script>
    <!-- 引入 Babel 以解析 JSX -->
    <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
    <!-- 引入 Lucide Icons -->
    <script src="https://unpkg.com/lucide@latest"></script>
</head>
<body>
    <div id="root"></div>

    <script type="text/babel">
        const { useState, useEffect, useRef } = React;

        // 自定義簡易的 Lucide 圖示元件 (為了在不用打包工具的環境下運作)
        const Icon = ({ name, size = 24, className = '', color = 'currentColor' }) => {
            const iconRef = useRef(null);
            useEffect(() => {
                if (iconRef.current && window.lucide) {
                    window.lucide.createIcons({
                        icons: { [name]: window.lucide.icons[name] },
                        nameAttr: 'data-lucide',
                        root: iconRef.current.parentNode
                    });
                }
            }, [name]);
            // 這裡使用一個 wrapper 來讓 lucide 替換
            return <span ref={iconRef}><i data-lucide={name} style={{ width: size, height: size, color }} className={className}></i></span>;
        };

        const Volume2 = (props) => <Icon name="Volume2" {...props} />;
        const CheckCircle = (props) => <Icon name="CheckCircle" {...props} />;
        const XCircle = (props) => <Icon name="XCircle" {...props} />;
        const RefreshCw = (props) => <Icon name="RefreshCw" {...props} />;
        const Star = (props) => <Icon name="Star" {...props} />;
        const User = (props) => <Icon name="User" {...props} />;
        const Trophy = (props) => <Icon name="Trophy" {...props} />;
        const Ear = (props) => <Icon name="Ear" {...props} />;

        // === 詞彙資料庫 ===
        const INITIAL_VOCABULARY = [
            { en: "environment", zh: "環境" },
            { en: "tends", zh: "往往" },
            { en: "strength", zh: "力氣" },
            { en: "maintain", zh: "維持" },
            { en: "routine", zh: "例行公事" }
        ];

        const CAPSULE_COLORS = [
            'bg-red-400', 'bg-blue-400', 'bg-green-400', 
            'bg-yellow-400', 'bg-purple-400', 'bg-pink-400', 'bg-orange-400'
        ];

        function GashaponGame() {
            const [gameState, setGameState] = useState('login'); 
            const [playerName, setPlayerName] = useState('');
            
            const [remainingQuestions, setRemainingQuestions] = useState([]); 
            const [currentWord, setCurrentWord] = useState(null);
            const [options, setOptions] = useState([]);
            const [optionType, setOptionType] = useState('zh'); 
            
            const [score, setScore] = useState(0);
            const [totalAttempts, setTotalAttempts] = useState(0);
            const [capsuleColor, setCapsuleColor] = useState('bg-red-400');
            const [feedback, setFeedback] = useState(null);
            const [knobRotation, setKnobRotation] = useState(0);

            useEffect(() => {
                window.speechSynthesis.getVoices();
            }, []);

            const playAudio = (text) => {
                if (!window.speechSynthesis) return;
                window.speechSynthesis.cancel();
                
                const utterance = new SpeechSynthesisUtterance(text);
                utterance.lang = 'en-US';
                utterance.rate = 0.85; 
                window.speechSynthesis.speak(utterance);
            };

            const initializeGame = () => {
                const pool = [];
                INITIAL_VOCABULARY.forEach(word => {
                    pool.push({ word, type: 'zh' }); 
                    pool.push({ word, type: 'en' }); 
                });
                return pool.sort(() => 0.5 - Math.random());
            };

            const handleStartGame = (e) => {
                e.preventDefault();
                if (!playerName.trim()) return;
                
                setScore(0);
                setTotalAttempts(0);
                setRemainingQuestions(initializeGame());
                setGameState('idle');
            };

            const generateQuestion = () => {
                const newRemaining = [...remainingQuestions];
                const nextQuestion = newRemaining.pop();
                setRemainingQuestions(newRemaining);

                const selectedWord = nextQuestion.word;
                const type = nextQuestion.type;
                setOptionType(type);
                
                const wrongWords = INITIAL_VOCABULARY.filter(w => w.en !== selectedWord.en);
                const shuffledWrong = wrongWords.sort(() => 0.5 - Math.random()).slice(0, 3);
                
                const allOptions = [...shuffledWrong.map(w => w[type]), selectedWord[type]];
                const finalOptions = allOptions.sort(() => 0.5 - Math.random());

                setCurrentWord(selectedWord);
                setOptions(finalOptions);
                setCapsuleColor(CAPSULE_COLORS[Math.floor(Math.random() * CAPSULE_COLORS.length)]);
            };

            const handleDraw = () => {
                if (gameState !== 'idle') return;

                setGameState('animating');
                setKnobRotation(prev => prev + 180);
                generateQuestion();

                setTimeout(() => {
                    setGameState('question');
                }, 1500);
            };

            useEffect(() => {
                if (gameState === 'question' && currentWord) {
                    playAudio(currentWord.en);
                }
            }, [gameState, currentWord]);

            const handleAnswer = (selectedOption) => {
                if (gameState !== 'question') return;

                const isCorrect = selectedOption === currentWord[optionType];
                setFeedback({ isCorrect, selectedOption });
                setGameState('feedback');
                setTotalAttempts(prev => prev + 1);

                if (isCorrect) {
                    setScore(prev => prev + 1);
                }

                setTimeout(() => {
                    if (remainingQuestions.length === 0) { 
                        setGameState('gameover');
                    } else {
                        setGameState('idle');
                    }
                    setFeedback(null);
                }, 2500);
            };

            const handleRestart = () => {
                setScore(0);
                setTotalAttempts(0);
                setRemainingQuestions(initializeGame());
                setGameState('idle');
            };

            return (
                <div className="min-h-screen bg-amber-50 flex flex-col items-center py-8 px-4 font-sans select-none">
                    <style>{`
                        @keyframes dropAndBounce {
                            0% { transform: translateY(-50px) scale(0.5); opacity: 0; }
                            60% { transform: translateY(20px) scale(1); opacity: 1; }
                            80% { transform: translateY(-5px) scale(1); }
                            100% { transform: translateY(0) scale(1); }
                        }
                        @keyframes shake {
                            0%, 100% { transform: rotate(0deg); }
                            25% { transform: rotate(-3deg); }
                            75% { transform: rotate(3deg); }
                        }
                        .animate-drop {
                            animation: dropAndBounce 0.6s cubic-bezier(0.175, 0.885, 0.32, 1.275) forwards;
                        }
                        .animate-shake {
                            animation: shake 0.3s ease-in-out infinite;
                        }
                        .animate-pulse-fast {
                            animation: pulse 1s cubic-bezier(0.4, 0, 0.6, 1) infinite;
                        }
                    `}</style>

                    {gameState === 'login' && (
                        <div className="w-full max-w-md bg-white p-8 rounded-3xl shadow-xl border-4 border-amber-200 mt-12 flex flex-col items-center animate-drop">
                            <div className="w-24 h-24 bg-amber-100 rounded-full flex items-center justify-center mb-6">
                                <Star className="text-amber-500 fill-current w-12 h-12" />
                            </div>
                            <h1 className="text-3xl font-black text-amber-700 mb-2">單字聽力扭蛋機</h1>
                            <p className="text-gray-500 mb-8">仔細聽發音，選出正確的字義或單字！</p>
                            
                            <form onSubmit={handleStartGame} className="w-full flex flex-col gap-4">
                                <div className="relative">
                                    <div className="absolute left-4 top-1/2 -translate-y-1/2 text-gray-400">
                                        <User size={24} />
                                    </div>
                                    <input 
                                        type="text" 
                                        placeholder="請輸入你的姓名..." 
                                        value={playerName}
                                        onChange={(e) => setPlayerName(e.target.value)}
                                        className="w-full pl-12 pr-4 py-4 rounded-xl border-2 border-gray-200 focus:border-amber-400 focus:outline-none text-lg font-bold transition-colors"
                                        required
                                    />
                                </div>
                                <button 
                                    type="submit"
                                    className="w-full bg-amber-500 hover:bg-amber-600 text-white font-bold py-4 rounded-xl text-xl shadow-[0_6px_0_#d97706] active:shadow-[0_0px_0_#d97706] active:translate-y-[6px] transition-all"
                                >
                                    開始測驗
                                </button>
                            </form>
                        </div>
                    )}

                    {gameState === 'gameover' && (
                        <div className="w-full max-w-md bg-white p-8 rounded-3xl shadow-xl border-4 border-amber-200 mt-12 flex flex-col items-center animate-drop text-center">
                            <div className="text-yellow-400 mb-4 drop-shadow-md">
                                <Trophy size={128} />
                            </div>
                            <h2 className="text-3xl font-black text-gray-800 mb-2">測驗完成！</h2>
                            <div className="bg-amber-100 px-6 py-2 rounded-full mb-6 text-xl font-bold text-amber-800">
                                玩家：{playerName}
                            </div>
                            
                            <div className="w-full bg-gray-50 rounded-2xl p-6 mb-8 border-2 border-gray-100">
                                <p className="text-gray-500 font-bold mb-2">你的最終得分</p>
                                <div className="text-6xl font-black text-amber-500 flex items-center justify-center gap-2">
                                    {score} <span className="text-3xl text-gray-400">/ 10</span>
                                </div>
                                <p className="mt-4 font-bold text-lg text-gray-600">
                                    {score === 10 ? '太厲害了！全部答對！🎉' : score >= 6 ? '表現不錯喔！繼續保持！👍' : '再多練習幾次，你會越來越棒的！💪'}
                                </p>
                            </div>

                            <button 
                                onClick={handleRestart}
                                className="w-full bg-blue-500 hover:bg-blue-600 text-white font-bold py-4 rounded-xl text-xl shadow-[0_6px_0_#2563eb] active:shadow-[0_0px_0_#2563eb] active:translate-y-[6px] transition-all flex items-center justify-center gap-2"
                            >
                                <RefreshCw size={24} /> 再玩一次
                            </button>
                        </div>
                    )}

                    {['idle', 'animating', 'question', 'feedback'].includes(gameState) && (
                        <>
                            <div className="w-full max-w-md flex justify-between items-center bg-white p-4 rounded-2xl shadow-sm mb-6 border-2 border-amber-200">
                                <div className="font-bold text-amber-700 flex items-center gap-2">
                                    <span className="text-amber-500"><User size={20} /></span> 
                                    {playerName}
                                </div>
                                <div className="font-bold text-gray-600 bg-amber-100 px-4 py-1 rounded-full border border-amber-200">
                                    得分: <span className="text-amber-600">{score}</span> / {totalAttempts}
                                </div>
                            </div>

                            <div className="w-full max-w-md flex flex-col items-center">
                                <div className={`relative flex flex-col items-center z-10 transition-transform duration-300 ${gameState === 'animating' ? 'animate-shake' : ''}`}>
                                    
                                    <div className="w-64 h-64 bg-cyan-50 rounded-t-full border-8 border-b-0 border-red-500 relative overflow-hidden flex flex-wrap-reverse justify-center content-start gap-1 p-4 shadow-inner">
                                        <div className="absolute inset-0 bg-white opacity-20 rounded-t-full z-10"></div>
                                        {CAPSULE_COLORS.map((color, i) => (
                                            <div key={i} className={`w-12 h-12 rounded-full ${color} opacity-80 border-2 border-black/10`} style={{ transform: `rotate(${Math.random() * 90}deg)` }}>
                                                <div className="w-full h-1/2 bg-white/30 rounded-t-full"></div>
                                            </div>
                                        ))}
                                        {CAPSULE_COLORS.map((color, i) => (
                                            <div key={`2-${i}`} className={`w-12 h-12 rounded-full ${color} opacity-80 border-2 border-black/10`} style={{ transform: `rotate(${Math.random() * 90}deg)` }}>
                                                <div className="w-full h-1/2 bg-white/30 rounded-t-full"></div>
                                            </div>
                                        ))}
                                    </div>

                                    <div className="w-72 h-56 bg-red-500 rounded-3xl rounded-t-none border-8 border-t-0 border-red-600 shadow-xl relative flex flex-col items-center pt-4 z-20">
                                        
                                        <div className="w-40 h-24 bg-red-400 rounded-xl shadow-inner border-2 border-red-700 flex items-center justify-center relative">
                                            <button 
                                                onClick={handleDraw}
                                                disabled={gameState !== 'idle'}
                                                className={`w-16 h-16 bg-gray-100 rounded-full border-4 border-gray-400 flex items-center justify-center shadow-md transition-all duration-300 ${gameState !== 'idle' ? 'opacity-80 cursor-not-allowed' : 'hover:bg-white hover:scale-105 active:scale-95 cursor-pointer'}`}
                                                style={{ transform: `rotate(${knobRotation}deg)` }}
                                                title="轉一下扭蛋"
                                            >
                                                <div className="w-3 h-10 bg-gray-400 rounded-full"></div>
                                            </button>
                                            <div className="absolute right-4 top-4 w-2 h-8 bg-gray-800 rounded-full"></div>
                                        </div>

                                        <div className="w-24 h-16 bg-gray-900 rounded-t-2xl absolute bottom-0 shadow-inner"></div>

                                        {gameState !== 'idle' && (
                                            <div className={`absolute bottom-[-20px] transition-all duration-500 z-30 flex items-center justify-center ${
                                                gameState === 'animating' 
                                                    ? 'w-16 h-16 animate-drop' 
                                                    : 'w-48 h-48 translate-y-24 shadow-2xl'
                                                }`}
                                            >
                                                <div className={`absolute inset-0 rounded-full ${capsuleColor} transition-all duration-500 border-4 border-white shadow-lg ${gameState === 'animating' ? '' : 'scale-110 opacity-0'}`}>
                                                    <div className="w-full h-1/2 bg-white/40 rounded-t-full"></div>
                                                </div>

                                                <div className={`bg-white border-4 border-amber-300 rounded-full w-[110%] h-[110%] flex flex-col items-center justify-center p-2 shadow-xl transition-all duration-500 ${gameState === 'animating' ? 'scale-0 opacity-0' : 'scale-100 opacity-100'}`}>
                                                    {currentWord && (
                                                        <button 
                                                            onClick={() => playAudio(currentWord.en)}
                                                            className="w-full h-full rounded-full bg-blue-50 text-blue-500 hover:bg-blue-100 hover:text-blue-600 flex flex-col items-center justify-center gap-2 transition-all group"
                                                            title="再聽一次"
                                                        >
                                                            <div className="bg-blue-500 text-white p-4 rounded-full group-hover:scale-110 transition-transform shadow-md animate-pulse-fast">
                                                                <Volume2 size={36} />
                                                            </div>
                                                            <span className="font-bold text-sm">聽發音</span>
                                                        </button>
                                                    )}
                                                </div>
                                            </div>
                                        )}
                                    </div>
                                </div>

                                <div className={`w-full mt-32 transition-all duration-500 ${gameState === 'idle' || gameState === 'animating' ? 'opacity-0 pointer-events-none translate-y-8' : 'opacity-100 translate-y-0'}`}>
                                    
                                    {gameState === 'feedback' && feedback ? (
                                        <div className={`w-full p-6 rounded-2xl text-center text-xl font-bold flex flex-col items-center gap-3 border-4 ${feedback.isCorrect ? 'bg-green-50 border-green-200 text-green-700' : 'bg-red-50 border-red-200 text-red-700'}`}>
                                            <div className="flex items-center gap-2">
                                                {feedback.isCorrect ? <CheckCircle size={32} /> : <XCircle size={32} />}
                                                <span>{feedback.isCorrect ? '答對囉！' : '哎呀，答錯了！'}</span>
                                            </div>
                                            <div className="bg-white px-6 py-3 rounded-xl shadow-sm border border-gray-100 mt-2 w-full flex flex-col items-center">
                                                <span className="text-gray-800 text-2xl tracking-wide">{currentWord?.en}</span>
                                                <span className="text-gray-500 text-lg mt-1">{currentWord?.zh}</span>
                                            </div>
                                        </div>
                                    ) : (
                                        <>
                                            <p className="text-center text-gray-500 font-bold mb-3 flex items-center justify-center gap-2">
                                                <Ear size={18} /> 請根據發音，選出正確的{optionType === 'zh' ? '中文意思' : '英文單字'}
                                            </p>
                                            <div className="grid grid-cols-2 gap-3">
                                                {options.map((opt, idx) => (
                                                    <button
                                                        key={idx}
                                                        onClick={() => handleAnswer(opt)}
                                                        className="bg-white border-2 border-gray-200 p-4 rounded-2xl text-xl font-bold text-gray-700 shadow-sm hover:shadow-md hover:border-blue-400 hover:text-blue-600 hover:bg-blue-50 active:scale-95 transition-all flex items-center justify-center min-h-[80px]"
                                                    >
                                                        {opt}
                                                    </button>
                                                ))}
                                            </div>
                                        </>
                                    )}
                                </div>

                                {gameState === 'idle' && (
                                    <div className="mt-8 text-center animate-pulse flex flex-col items-center text-amber-700 font-bold bg-amber-100 px-6 py-2 rounded-full border border-amber-200">
                                        <p>點擊上方機台的【灰色旋鈕】來轉扭蛋！</p>
                                        <p className="text-sm opacity-80 mt-1">還有 {remainingQuestions.length} 顆單字扭蛋</p>
                                    </div>
                                )}
                            </div>
                        </>
                    )}
                </div>
            );
        }

        const root = ReactDOM.createRoot(document.getElementById('root'));
        root.render(<GashaponGame />);
    </script>
</body>
</html>
