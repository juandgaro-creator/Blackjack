import React, { useState, useEffect, useCallback } from 'react';
import { Trophy, History, RotateCcw, Play, Plus, hand, CreditCard, ChevronDown, ChevronUp } from 'lucide-react';

// --- Constants & Utilities ---
const SUITS = ['♠', '♣', '♥', '♦'];
const VALUES = ['A', '2', '3', '4', '5', '6', '7', '8', '9', '10', 'J', 'Q', 'K'];

const createDeck = () => {
  const deck = [];
  for (const suit of SUITS) {
    for (const value of VALUES) {
      deck.push({ suit, value });
    }
  }
  return deck.sort(() => Math.random() - 0.5);
};

const getCardValue = (card) => {
  if (['J', 'Q', 'K'].includes(card.value)) return 10;
  if (card.value === 'A') return 11;
  return parseInt(card.value);
};

const calculateScore = (hand) => {
  let score = hand.reduce((sum, card) => sum + getCardValue(card), 0);
  let aces = hand.filter(card => card.value === 'A').length;
  
  while (score > 21 && aces > 0) {
    score -= 10;
    aces -= 1;
  }
  return score;
};

// --- Components ---

const Card = ({ card, hidden }) => {
  const isRed = card?.suit === '♥' || card?.suit === '♦';
  
  if (hidden) {
    return (
      <div className="w-16 h-24 md:w-20 md:h-32 bg-blue-800 border-2 border-white rounded-lg shadow-lg flex items-center justify-center">
        <div className="w-12 h-20 md:w-16 md:h-28 border border-blue-600 rounded flex items-center justify-center opacity-50">
          <div className="text-white text-xl">?</div>
        </div>
      </div>
    );
  }

  return (
    <div className={`w-16 h-24 md:w-20 md:h-32 bg-white border-2 border-gray-200 rounded-lg shadow-lg flex flex-col p-2 justify-between ${isRed ? 'text-red-600' : 'text-gray-900'}`}>
      <div className="text-sm md:text-lg font-bold leading-none">{card.value}</div>
      <div className="text-2xl md:text-4xl self-center">{card.suit}</div>
      <div className="text-sm md:text-lg font-bold leading-none self-end rotate-180">{card.value}</div>
    </div>
  );
};

export default function App() {
  const [deck, setDeck] = useState([]);
  const [playerHand, setPlayerHand] = useState([]);
  const [dealerHand, setDealerHand] = useState([]);
  const [gameState, setGameState] = useState('betting'); // betting, playing, dealer_turn, ended
  const [message, setMessage] = useState('¡Haz tu apuesta!');
  const [balance, setBalance] = useState(1000);
  const [bet, setBet] = useState(10);
  const [history, setHistory] = useState([]);
  const [showHistory, setShowHistory] = useState(false);

  // Initialize deck
  useEffect(() => {
    setDeck(createDeck());
  }, []);

  const addToHistory = (result, playerFinalScore, dealerFinalScore) => {
    const entry = {
      id: Date.now(),
      result,
      playerScore: playerFinalScore,
      dealerScore: dealerFinalScore,
      bet,
      balanceAfter: balance + (result === 'Ganaste' ? bet : result === 'Perdiste' ? -bet : 0),
      timestamp: new Date().toLocaleTimeString()
    };
    setHistory(prev => [entry, ...prev]);
  };

  const dealInitialCards = () => {
    if (balance < bet) {
      setMessage('Saldo insuficiente');
      return;
    }

    const newDeck = createDeck();
    const pHand = [newDeck.pop(), newDeck.pop()];
    const dHand = [newDeck.pop(), newDeck.pop()];
    
    setPlayerHand(pHand);
    setDealerHand(dHand);
    setDeck(newDeck);
    setGameState('playing');
    setMessage('¿Pedir o Plantarse?');

    // Check for natural blackjack
    if (calculateScore(pHand) === 21) {
      handleStand(pHand, dHand, newDeck);
    }
  };

  const handleHit = () => {
    const newDeck = [...deck];
    const newHand = [...playerHand, newDeck.pop()];
    setPlayerHand(newHand);
    setDeck(newDeck);

    if (calculateScore(newHand) > 21) {
      endGame('Perdiste', newHand, dealerHand);
    }
  };

  const handleStand = (currentPHand = playerHand, currentDHand = dealerHand, currentDeck = deck) => {
    setGameState('dealer_turn');
    let tempDealerHand = [...currentDHand];
    let tempDeck = [...currentDeck];

    // Dealer logic: hit until 17
    while (calculateScore(tempDealerHand) < 17) {
      tempDealerHand.push(tempDeck.pop());
    }

    setDealerHand(tempDealerHand);
    setDeck(tempDeck);

    const pScore = calculateScore(currentPHand);
    const dScore = calculateScore(tempDealerHand);

    if (dScore > 21 || pScore > dScore) {
      endGame('Ganaste', currentPHand, tempDealerHand);
    } else if (pScore < dScore) {
      endGame('Perdiste', currentPHand, tempDealerHand);
    } else {
      endGame('Empate', currentPHand, tempDealerHand);
    }
  };

  const endGame = (result, pHand, dHand) => {
    const pScore = calculateScore(pHand);
    const dScore = calculateScore(dHand);
    
    setGameState('ended');
    setMessage(result === 'Ganaste' ? '🎉 ¡Has ganado!' : result === 'Perdiste' ? '❌ El crupier gana' : '🤝 Empate');
    
    if (result === 'Ganaste') setBalance(prev => prev + bet);
    if (result === 'Perdiste') setBalance(prev => prev - bet);
    
    addToHistory(result, pScore, dScore);
  };

  const resetGame = () => {
    setGameState('betting');
    setPlayerHand([]);
    setDealerHand([]);
    setMessage('¡Haz tu apuesta!');
  };

  return (
    <div className="min-h-screen bg-green-900 text-white font-sans p-4 md:p-8">
      <div className="max-w-4xl mx-auto">
        {/* Header Stats */}
        <div className="flex justify-between items-center mb-8 bg-black/30 p-4 rounded-xl backdrop-blur-sm">
          <div className="flex items-center gap-3">
            <div className="bg-yellow-500 p-2 rounded-lg text-black">
              <CreditCard size={24} />
            </div>
            <div>
              <p className="text-xs uppercase opacity-70">Tu Saldo</p>
              <p className="text-xl font-bold font-mono">${balance}</p>
            </div>
          </div>
          <button 
            onClick={() => setShowHistory(!showHistory)}
            className="flex items-center gap-2 bg-white/10 hover:bg-white/20 px-4 py-2 rounded-lg transition-colors"
          >
            <History size={20} />
            <span className="hidden sm:inline">Historial</span>
          </button>
        </div>

        {/* Main Game Table */}
        <div className="relative bg-green-800 border-8 border-yellow-800 rounded-[50px] min-h-[500px] shadow-2xl p-6 flex flex-col justify-between overflow-hidden">
          <div className="absolute inset-0 opacity-10 pointer-events-none flex items-center justify-center">
             <Trophy size={300} />
          </div>

          {/* Dealer Area */}
          <div className="flex flex-col items-center gap-4 z-10">
            <div className="bg-black/20 px-4 py-1 rounded-full text-sm">
              Crupier: {gameState === 'playing' ? '?' : calculateScore(dealerHand)}
            </div>
            <div className="flex gap-2 min-h-[128px]">
              {dealerHand.map((card, i) => (
                <Card key={i} card={card} hidden={i === 1 && gameState === 'playing'} />
              ))}
              {dealerHand.length === 0 && <div className="w-20 h-32 border-2 border-white/20 border-dashed rounded-lg" />}
            </div>
          </div>

          {/* Message Area */}
          <div className="text-center z-10 my-4">
            <h2 className="text-2xl md:text-3xl font-bold tracking-tight drop-shadow-md">{message}</h2>
          </div>

          {/* Player Area */}
          <div className="flex flex-col items-center gap-4 z-10">
            <div className="flex gap-2 min-h-[128px] justify-center flex-wrap">
              {playerHand.map((card, i) => (
                <Card key={i} card={card} />
              ))}
              {playerHand.length === 0 && <div className="w-20 h-32 border-2 border-white/20 border-dashed rounded-lg" />}
            </div>
            <div className="bg-black/20 px-4 py-1 rounded-full text-sm">
              Tu Puntuación: {calculateScore(playerHand)}
            </div>
          </div>
        </div>

        {/* Controls */}
        <div className="mt-8 flex justify-center gap-4">
          {gameState === 'betting' && (
            <div className="flex flex-col items-center gap-4 w-full max-w-sm">
              <div className="flex items-center gap-4 bg-black/40 p-2 rounded-2xl w-full justify-between">
                <button 
                  onClick={() => setBet(Math.max(10, bet - 10))}
                  className="w-12 h-12 flex items-center justify-center bg-red-600 rounded-xl hover:bg-red-500 active:scale-95 transition-all"
                >-</button>
                <div className="text-center">
                  <span className="text-xs uppercase opacity-60 block">Apuesta</span>
                  <span className="text-2xl font-bold font-mono">${bet}</span>
                </div>
                <button 
                   onClick={() => setBet(Math.min(balance, bet + 10))}
                   className="w-12 h-12 flex items-center justify-center bg-blue-600 rounded-xl hover:bg-blue-500 active:scale-95 transition-all"
                >+</button>
              </div>
              <button 
                onClick={dealInitialCards}
                className="w-full bg-yellow-500 hover:bg-yellow-400 text-black font-black py-4 rounded-2xl text-xl shadow-lg flex items-center justify-center gap-2 active:translate-y-1 transition-all"
              >
                <Play fill="black" size={24} /> REPARTIR
              </button>
            </div>
          )}

          {gameState === 'playing' && (
            <div className="flex gap-4 w-full max-w-sm">
              <button 
                onClick={handleHit}
                className="flex-1 bg-blue-600 hover:bg-blue-500 py-4 rounded-2xl font-bold text-xl shadow-lg active:scale-95 transition-all"
              >
                PEDIR
              </button>
              <button 
                onClick={() => handleStand()}
                className="flex-1 bg-red-600 hover:bg-red-500 py-4 rounded-2xl font-bold text-xl shadow-lg active:scale-95 transition-all"
              >
                PLANTARSE
              </button>
            </div>
          )}

          {gameState === 'ended' && (
            <button 
              onClick={resetGame}
              className="w-full max-w-sm bg-yellow-500 hover:bg-yellow-400 text-black font-black py-4 rounded-2xl text-xl shadow-lg flex items-center justify-center gap-2 active:scale-95 transition-all"
            >
              <RotateCcw size={24} /> NUEVA MANO
            </button>
          )}
        </div>

        {/* History Modal Overlay */}
        {showHistory && (
          <div className="fixed inset-0 bg-black/80 backdrop-blur-md z-50 p-4 flex items-center justify-center">
            <div className="bg-gray-900 w-full max-w-2xl max-h-[80vh] rounded-3xl overflow-hidden flex flex-col border border-white/10 shadow-2xl">
              <div className="p-6 border-b border-white/10 flex justify-between items-center bg-gray-800">
                <h3 className="text-xl font-bold flex items-center gap-2">
                  <History className="text-yellow-500" /> Registro de Partidas
                </h3>
                <button 
                  onClick={() => setShowHistory(false)}
                  className="text-gray-400 hover:text-white"
                >Cerrar</button>
              </div>
              <div className="overflow-y-auto p-4 space-y-3">
                {history.length === 0 ? (
                  <div className="text-center py-12 text-gray-500 italic">No hay partidas registradas aún</div>
                ) : (
                  history.map((game) => (
                    <div key={game.id} className="bg-white/5 rounded-xl p-4 flex items-center justify-between border border-white/5">
                      <div className="flex flex-col">
                        <span className={`text-sm font-bold uppercase ${
                          game.result === 'Ganaste' ? 'text-green-400' : 
                          game.result === 'Perdiste' ? 'text-red-400' : 'text-yellow-400'
                        }`}>
                          {game.result}
                        </span>
                        <span className="text-xs text-gray-400">{game.timestamp}</span>
                      </div>
                      <div className="flex gap-4 text-center">
                        <div>
                          <p className="text-[10px] text-gray-500 uppercase">Tú</p>
                          <p className="font-mono">{game.playerScore}</p>
                        </div>
                        <div className="text-gray-600 self-center">vs</div>
                        <div>
                          <p className="text-[10px] text-gray-500 uppercase">Crupier</p>
                          <p className="font-mono">{game.dealerScore}</p>
                        </div>
                      </div>
                      <div className="text-right">
                        <p className="text-xs text-gray-500 italic">Apuesta: ${game.bet}</p>
                        <p className="font-bold text-yellow-500">${game.balanceAfter}</p>
                      </div>
                    </div>
                  ))
                )}
              </div>
            </div>
          </div>
        )}
      </div>
    </div>
  );
}