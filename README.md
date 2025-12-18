# Co-Creation-Culture---SPARK

import React, { useState, useEffect, useCallback, useRef } from 'react';
import { generateScenario } from './services/geminiService';
import { Scenario, GameState, DialogueChoice, ChoiceQuality } from './types';
import Dashboard from './components/Dashboard';
import Stage from './components/Stage';

const INITIAL_STATE: GameState = {
  heart: 80,
  sop: 100,
  timer: 300,
  currentScenarioIndex: 0,
  history: [],
  isGameOver: false,
  isStarted: false,
};

const App: React.FC = () => {
  const [state, setState] = useState<GameState>(INITIAL_STATE);
  const [currentScenario, setCurrentScenario] = useState<Scenario | null>(null);
  const [displayedDialogue, setDisplayedDialogue] = useState("");
  const [isTyping, setIsTyping] = useState(false);
  const [isLoading, setIsLoading] = useState(false);
  const [feedback, setFeedback] = useState<{ text: string; quality: ChoiceQuality } | null>(null);
  const [mood, setMood] = useState<'neutral' | 'happy' | 'annoyed' | 'angry'>('neutral');
  
  const typingTimerRef = useRef<number | null>(null);

  const typeText = useCallback((text: string) => {
    if (typingTimerRef.current) clearInterval(typingTimerRef.current);
    setDisplayedDialogue("");
    setIsTyping(true);
    let i = 0;
    typingTimerRef.current = window.setInterval(() => {
      setDisplayedDialogue((prev) => prev + text.charAt(i));
      i++;
      if (i >= text.length) {
        if (typingTimerRef.current) clearInterval(typingTimerRef.current);
        setIsTyping(false);
      }
    }, 30);
  }, []);

  const startNewScenario = useCallback(async () => {
    setIsLoading(true);
    setFeedback(null);
    setMood('neutral');
    try {
      const scenario = await generateScenario();
      setCurrentScenario(scenario);
      typeText(scenario.situation);
    } catch (error) {
      console.error("Failed to load scenario:", error);
    } finally {
      setIsLoading(false);
    }
  }, [typeText]);

  const handleStartGame = () => {
    setState(prev => ({ ...prev, isStarted: true }));
    startNewScenario();
  };

  const handleChoice = (choice: DialogueChoice) => {
    let newMood: 'happy' | 'annoyed' | 'angry' = 'annoyed';
    if (choice.quality === ChoiceQuality.BEST) newMood = 'happy';
    if (choice.quality === ChoiceQuality.IMPROVEMENT) newMood = 'angry';

    setMood(newMood);
    setFeedback({ text: choice.feedback, quality: choice.quality });
    typeText(choice.feedback);

    setState(prev => {
      const newHeart = Math.min(100, Math.max(0, prev.heart + choice.heartImpact));
      const newSop = Math.min(100, Math.max(0, prev.sop + choice.sopImpact));
      const newIndex = prev.currentScenarioIndex + 1;
      const isOver = newIndex >= 5 || newHeart <= 0 || newSop <= 0;

      return {
        ...prev,
        heart: newHeart,
        sop: newSop,
        currentScenarioIndex: newIndex,
        history: [...prev.history, { scenarioId: currentScenario?.id || '', choice }],
        isGameOver: isOver
      };
    });
  };

  useEffect(() => {
    let interval: number;
    if (state.isStarted && !state.isGameOver && !isLoading) {
      interval = window.setInterval(() => {
        setState(prev => {
          if (prev.timer <= 0) return { ...prev, isGameOver: true };
          return { ...prev, timer: prev.timer - 1 };
        });
      }, 1000);
    }
    return () => clearInterval(interval);
  }, [state.isStarted, state.isGameOver, isLoading]);

  if (!state.isStarted) {
    return (
      <div className="min-h-screen flex flex-col items-center justify-center bg-slate-950 text-white p-6 relative overflow-hidden">
        <div className="absolute inset-0 z-0 opacity-20">
          <div className="absolute top-0 left-0 w-full h-full bg-[radial-gradient(circle_at_center,_var(--tw-gradient-stops))] from-red-900/40 via-transparent to-transparent"></div>
        </div>

        <div className="max-w-md w-full text-center space-y-8 z-10">
          <div className="inline-block p-6 bg-red-600 rounded-[2rem] shadow-[0_0_50px_rgba(220,38,38,0.3)] mb-4 animate-bounce">
            <i className="fas fa-shield-heart text-5xl"></i>
          </div>
          <div>
            <h1 className="text-6xl font-black tracking-tighter mb-2 italic">SERVICE HERO</h1>
            <div className="h-1 w-24 bg-red-600 mx-auto"></div>
          </div>
          <p className="text-slate-400 text-lg font-medium leading-relaxed">
            มาตรฐานคือ <span className="text-white">ระเบียบ</span> <br/>การบริการคือ <span className="text-red-500">หัวใจ</span> <br/>คุณจะรักษาสมดุลได้หรือไม่?
          </p>
          <div className="space-y-4 pt-8">
            <button 
              onClick={handleStartGame}
              className="w-full py-5 bg-red-600 hover:bg-red-700 text-white rounded-2xl font-black text-2xl transition-all shadow-xl hover:shadow-red-900/40 transform hover:-translate-y-1 active:scale-95 flex items-center justify-center space-x-3"
            >
              <span>เริ่มภารกิจ</span>
              <i className="fas fa-chevron-right text-sm"></i>
            </button>
            <div className="flex justify-center space-x-4 text-[10px] text-slate-500 font-bold uppercase tracking-widest">
              <span>ความเป๊ะตาม SOP</span>
              <span>•</span>
              <span>SERVICE MIND</span>
            </div>
          </div>
        </div>
      </div>
    );
  }

  if (state.isGameOver) {
    const score = Math.floor((state.heart + state.sop) / 2);
    return (
      <div className="min-h-screen bg-slate-950 flex flex-col items-center justify-center p-6">
        <div className="bg-white rounded-[3rem] shadow-2xl p-12 max-w-2xl w-full text-center border-b-8 border-slate-200">
          <div className="w-24 h-24 bg-red-600 text-white rounded-3xl rotate-12 flex items-center justify-center mx-auto mb-8 shadow-2xl">
            <i className={`fas ${score > 70 ? 'fa-crown' : 'fa-clipboard-check'} text-4xl`}></i>
          </div>
          <h2 className="text-5xl font-black text-slate-900 mb-2 tracking-tighter">สรุปผลภารกิจ</h2>
          <p className="text-slate-500 font-medium mb-10">การประเมินประสิทธิภาพการบริการเสร็จสิ้น</p>
          
          <div className="grid grid-cols-2 gap-4 mb-10">
            <div className="p-6 bg-slate-50 rounded-3xl text-left border border-slate-100">
              <div className="text-slate-400 text-[10px] font-black uppercase mb-1">ใจลูกค้า (Heart)</div>
              <div className="text-4xl font-black text-red-600">{state.heart}%</div>
            </div>
            <div className="p-6 bg-slate-900 rounded-3xl text-left shadow-xl">
              <div className="text-slate-500 text-[10px] font-black uppercase mb-1">กฎระเบียบ (SOP)</div>
              <div className="text-4xl font-black text-white">{state.sop}%</div>
            </div>
          </div>

          <button 
            onClick={() => window.location.reload()}
            className="w-full py-5 bg-red-600 text-white rounded-2xl font-black text-xl transition-all hover:bg-red-700 shadow-xl"
          >
            เริ่มภารกิจใหม่
          </button>
        </div>
      </div>
    );
  }

  return (
    <div className="min-h-screen bg-slate-50 flex flex-col font-sans">
      <Dashboard heart={state.heart} sop={state.sop} timer={state.timer} />

      <main className="flex-grow max-w-6xl mx-auto w-full p-4 lg:p-10 flex flex-col space-y-8">
        {isLoading ? (
          <div className="flex-grow flex flex-col items-center justify-center space-y-6">
            <div className="relative w-24 h-24">
              <div className="absolute inset-0 border-8 border-slate-200 rounded-full"></div>
              <div className="absolute inset-0 border-8 border-red-600 border-t-transparent rounded-full animate-spin"></div>
            </div>
            <div className="text-center">
              <h3 className="text-2xl font-black text-slate-900 tracking-tight">กำลังวิเคราะห์สถานการณ์</h3>
              <p className="text-slate-500 font-medium animate-pulse">ดึงข้อมูลลูกค้าแบบเรียลไทม์...</p>
            </div>
          </div>
        ) : currentScenario ? (
          <div className="grid lg:grid-cols-5 gap-10 animate-fade-in">
            <div className="lg:col-span-3 space-y-6">
               <Stage 
                mood={mood} 
                customerName={currentScenario.customerName} 
                dialogue={displayedDialogue} 
                isTyping={isTyping}
              />
              
              {!feedback && !isTyping && (
                <div className="bg-white p-6 rounded-3xl shadow-sm border border-slate-100 flex items-start space-x-4">
                  <div className="w-10 h-10 bg-slate-900 rounded-2xl flex items-center justify-center shrink-0">
                    <i className="fas fa-info text-white text-xs"></i>
                  </div>
                  <div>
                    <h4 className="text-[10px] font-black text-slate-400 uppercase tracking-widest mb-1">ข้อมูลประกอบการตัดสินใจ</h4>
                    <p className="text-slate-600 text-sm italic">"{currentScenario.customerProfile}"</p>
                  </div>
                </div>
              )}
            </div>

            <div className="lg:col-span-2 flex flex-col justify-center space-y-6">
              {!feedback ? (
                <div className="space-y-4">
                  <div className="flex items-center space-x-3 mb-2 px-2">
                    <div className="h-px flex-grow bg-slate-200"></div>
                    <span className="text-[10px] font-black text-slate-400 uppercase tracking-widest whitespace-nowrap">เลือกการตอบสนองของคุณ</span>
                    <div className="h-px flex-grow bg-slate-200"></div>
                  </div>
                  
                  {currentScenario.choices.map((choice, idx) => (
                    <button
                      key={idx}
                      disabled={isTyping}
                      onClick={() => handleChoice(choice)}
                      className={`w-full text-left p-6 bg-white hover:bg-slate-50 border-2 border-slate-100 hover:border-red-600 rounded-3xl transition-all group flex items-start space-x-4 shadow-sm hover:shadow-xl ${isTyping ? 'opacity-50 cursor-not-allowed' : ''}`}
                    >
                      <div className="w-10 h-10 rounded-2xl bg-slate-100 flex items-center justify-center text-slate-900 group-hover:bg-red-600 group-hover:text-white font-black transition-colors shrink-0">
                        {String.fromCharCode(65 + idx)}
                      </div>
                      <span className="text-slate-700 font-bold leading-tight pt-2">{choice.text}</span>
                    </button>
                  ))}
                </div>
              ) : (
                <div className="space-y-6">
                  <div className={`p-8 rounded-[2.5rem] shadow-2xl border-4 transition-all animate-bounce-subtle ${
                    feedback.quality === ChoiceQuality.BEST ? 'bg-white border-green-500' :
                    feedback.quality === ChoiceQuality.STANDARD ? 'bg-white border-amber-500' : 'bg-white border-red-600'
                  }`}>
                    <div className="flex items-center space-x-3 mb-4">
                       <span className={`px-3 py-1 rounded-full text-[10px] font-black text-white uppercase ${
                        feedback.quality === ChoiceQuality.BEST ? 'bg-green-500' :
                        feedback.quality === ChoiceQuality.STANDARD ? 'bg-amber-500' : 'bg-red-600'
                      }`}>
                        {feedback.quality === ChoiceQuality.BEST ? 'ดีเยี่ยม' : feedback.quality === ChoiceQuality.STANDARD ? 'มาตรฐาน' : 'ต้องปรับปรุง'}
                      </span>
                    </div>
                    <p className="text-slate-700 font-medium text-lg italic leading-relaxed">
                      วิเคราะห์ผล: {feedback.text}
                    </p>
                  </div>
                  
                  <button
                    onClick={startNewScenario}
                    disabled={isTyping}
                    className="w-full py-5 bg-slate-900 text-white rounded-2xl font-black text-xl shadow-2xl hover:bg-black transition-all flex items-center justify-center space-x-3"
                  >
                    <span>เหตุการณ์ถัดไป</span>
                    <i className="fas fa-arrow-right"></i>
                  </button>
                </div>
              )}
            </div>
          </div>
        ) : null}
      </main>

      <footer className="p-6 bg-white border-t border-slate-100 flex justify-between items-center">
        <div className="flex items-center space-x-4">
          <div className="flex -space-x-2">
            {[...Array(5)].map((_, i) => (
              <div key={i} className={`w-3 h-3 rounded-full border-2 border-white ${i < state.currentScenarioIndex ? 'bg-red-600' : 'bg-slate-200'}`}></div>
            ))}
          </div>
          <span className="text-[10px] font-black text-slate-400 uppercase tracking-widest">
            สถานการณ์ที่ {state.currentScenarioIndex + 1} จาก 5
          </span>
        </div>
        <div className="text-[10px] font-black text-slate-300 uppercase tracking-widest">
          โมดูลฝึกอบรมการบริการระดับมาตรฐาน • เวอร์ชั่นเบต้า
        </div>
      </footer>
    </div>
  );
};

export default App;
