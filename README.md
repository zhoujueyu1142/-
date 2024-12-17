import React, { useState, useEffect, useCallback } from 'react';

const BOARD_WIDTH = 10;
const BOARD_HEIGHT = 20;
const SHAPES = {
  I: [[1, 1, 1, 1]],
  L: [[1, 0], [1, 0], [1, 1]],
  J: [[0, 1], [0, 1], [1, 1]],
  O: [[1, 1], [1, 1]],
  Z: [[1, 1, 0], [0, 1, 1]],
  S: [[0, 1, 1], [1, 1, 0]],
  T: [[1, 1, 1], [0, 1, 0]]
};

const createEmptyBoard = () => 
  Array(BOARD_HEIGHT).fill(null).map(() => Array(BOARD_WIDTH).fill(0));

const TetrisGame = () => {
  const [board, setBoard] = useState(createEmptyBoard());
  const [currentPiece, setCurrentPiece] = useState(null);
  const [position, setPosition] = useState({ x: 0, y: 0 });
  const [score, setScore] = useState(0);
  const [gameOver, setGameOver] = useState(false);
  const [isPaused, setIsPaused] = useState(false);
  const [nextPiece, setNextPiece] = useState(null);

  const getRandomPiece = useCallback(() => {
    const shapes = Object.keys(SHAPES);
    const shape = shapes[Math.floor(Math.random() * shapes.length)];
    return SHAPES[shape];
  }, []);

  const initGame = useCallback(() => {
    setBoard(createEmptyBoard());
    setCurrentPiece(getRandomPiece());
    setNextPiece(getRandomPiece());
    setPosition({ x: Math.floor(BOARD_WIDTH / 2) - 1, y: 0 });
    setScore(0);
    setGameOver(false);
    setIsPaused(false);
  }, [getRandomPiece]);

  const checkCollision = useCallback((piece, newPos) => {
    if (!piece) return false;
    
    for (let y = 0; y < piece.length; y++) {
      for (let x = 0; x < piece[y].length; x++) {
        if (piece[y][x]) {
          const newX = newPos.x + x;
          const newY = newPos.y + y;
          
          if (newX < 0 || newX >= BOARD_WIDTH || 
              newY >= BOARD_HEIGHT ||
              (newY >= 0 && board[newY][newX])) {
            return true;
          }
        }
      }
    }
    return false;
  }, [board]);

  const rotatePiece = useCallback((piece) => {
    if (!piece) return piece;
    const newPiece = Array(piece[0].length).fill(null)
      .map(() => Array(piece.length).fill(0));
    
    for (let y = 0; y < piece.length; y++) {
      for (let x = 0; x < piece[y].length; x++) {
        newPiece[x][piece.length - 1 - y] = piece[y][x];
      }
    }
    return newPiece;
  }, []);

  const mergePieceToBoard = useCallback(() => {
    if (!currentPiece) return;
    
    const newBoard = board.map(row => [...row]);
    for (let y = 0; y < currentPiece.length; y++) {
      for (let x = 0; x < currentPiece[y].length; x++) {
        if (currentPiece[y][x]) {
          const boardY = position.y + y;
          if (boardY < 0) {
            setGameOver(true);
            return;
          }
          newBoard[boardY][position.x + x] = 1;
        }
      }
    }
    
    // Check for completed lines
    let linesCleared = 0;
    for (let y = BOARD_HEIGHT - 1; y >= 0; y--) {
      if (newBoard[y].every(cell => cell === 1)) {
        newBoard.splice(y, 1);
        newBoard.unshift(Array(BOARD_WIDTH).fill(0));
        linesCleared++;
        y++; // Recheck the same line
      }
    }
    
    setScore(prev => prev + linesCleared * 100);
    setBoard(newBoard);
    setCurrentPiece(nextPiece);
    setNextPiece(getRandomPiece());
    setPosition({ x: Math.floor(BOARD_WIDTH / 2) - 1, y: 0 });
  }, [board, currentPiece, nextPiece, position, getRandomPiece]);

  const moveDown = useCallback(() => {
    if (gameOver || isPaused) return;
    
    const newPos = { ...position, y: position.y + 1 };
    if (checkCollision(currentPiece, newPos)) {
      mergePieceToBoard();
    } else {
      setPosition(newPos);
    }
  }, [position, currentPiece, checkCollision, mergePieceToBoard, gameOver, isPaused]);

  const moveHorizontal = useCallback((direction) => {
    if (gameOver || isPaused) return;
    
    const newPos = { ...position, x: position.x + direction };
    if (!checkCollision(currentPiece, newPos)) {
      setPosition(newPos);
    }
  }, [position, currentPiece, checkCollision, gameOver, isPaused]);

  const rotate = useCallback(() => {
    if (gameOver || isPaused) return;
    
    const rotatedPiece = rotatePiece(currentPiece);
    if (!checkCollision(rotatedPiece, position)) {
      setCurrentPiece(rotatedPiece);
    }
  }, [currentPiece, position, checkCollision, rotatePiece, gameOver, isPaused]);

  useEffect(() => {
    const handleKeyPress = (e) => {
      switch (e.key) {
        case 'ArrowLeft':
          moveHorizontal(-1);
          break;
        case 'ArrowRight':
          moveHorizontal(1);
          break;
        case 'ArrowDown':
          moveDown();
          break;
        case 'ArrowUp':
          rotate();
          break;
        case ' ':
          setIsPaused(prev => !prev);
          break;
        default:
          break;
      }
    };

    window.addEventListener('keydown', handleKeyPress);
    return () => window.removeEventListener('keydown', handleKeyPress);
  }, [moveHorizontal, moveDown, rotate]);

  useEffect(() => {
    if (!gameOver && !isPaused) {
      const interval = setInterval(moveDown, 1000);
      return () => clearInterval(interval);
    }
  }, [moveDown, gameOver, isPaused]);

  useEffect(() => {
    initGame();
  }, [initGame]);

  return (
    <div className="flex flex-col items-center p-4 bg-gray-100 min-h-screen">
      <div className="flex gap-8">
        <div className="bg-white p-4 rounded-lg shadow-lg">
          <div className="grid grid-cols-10 gap-px bg-gray-200 p-1">
            {board.map((row, y) => (
              row.map((cell, x) => {
                const isPiecePart = currentPiece && 
                  y >= position.y && 
                  y < position.y + currentPiece.length &&
                  x >= position.x && 
                  x < position.x + currentPiece[0].length &&
                  currentPiece[y - position.y][x - position.x];
                
                return (
                  <div
                    key={`${y}-${x}`}
                    className={`w-6 h-6 ${
                      cell || isPiecePart 
                        ? 'bg-blue-500' 
                        : 'bg-gray-100'
                    }`}
                  />
                );
              })
            ))}
          </div>
        </div>

        <div className="flex flex-col gap-4">
          <div className="bg-white p-4 rounded-lg shadow-lg">
            <h2 className="text-xl font-bold mb-2">下一个方块</h2>
            <div className="grid grid-cols-4 gap-px bg-gray-200 p-1">
              {Array(4).fill(null).map((_, y) => (
                Array(4).fill(null).map((_, x) => (
                  <div
                    key={`next-${y}-${x}`}
                    className={`w-6 h-6 ${
                      nextPiece && 
                      y < nextPiece.length && 
                      x < nextPiece[0].length && 
                      nextPiece[y][x]
                        ? 'bg-blue-500'
                        : 'bg-gray-100'
                    }`}
                  />
                ))
              ))}
            </div>
          </div>

          <div className="bg-white p-4 rounded-lg shadow-lg">
            <h2 className="text-xl font-bold mb-2">分数: {score}</h2>
            <button
              onClick={() => initGame()}
              className="w-full bg-blue-500 text-white py-2 px-4 rounded hover:bg-blue-600 mb-2"
            >
              新游戏
            </button>
            <button
              onClick={() => setIsPaused(p => !p)}
              className="w-full bg-green-500 text-white py-2 px-4 rounded hover:bg-green-600"
            >
              {isPaused ? '继续' : '暂停'}
            </button>
          </div>

          <div className="bg-white p-4 rounded-lg shadow-lg">
            <h3 className="font-bold mb-2">操作说明：</h3>
            <ul className="text-sm">
              <li>← →: 左右移动</li>
              <li>↓: 加速下落</li>
              <li>↑: 旋转</li>
              <li>空格: 暂停/继续</li>
            </ul>
          </div>
        </div>
      </div>

      {gameOver && (
        <div className="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center">
          <div className="bg-white p-8 rounded-lg text-center">
            <h2 className="text-2xl font-bold mb-4">游戏结束!</h2>
            <p className="mb-4">最终分数: {score}</p>
            <button
              onClick={initGame}
              className="bg-blue-500 text-white py-2 px-4 rounded hover:bg-blue-600"
            >
              再玩一次
            </button>
          </div>
        </div>
      )}
    </div>
  );
};

export default TetrisGame;# -
小游戏
