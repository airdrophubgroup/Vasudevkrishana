# // inside your staking / claim screen
const playGame = async () => {
  try {
    const res = await fetch(`${API}/api/game/play`, {
      method: 'POST',
      headers: {'Content-Type': 'application/json'},
      body: JSON.stringify({wallet_address: userWallet})
    });
    const data = await res.json();
    Alert.alert('Game result', data.message);
    // refresh balances / staking screen
  } catch (e) {
    Alert.alert('Error', e.message);
  }
};

<Button title="🎮  Play Daily Coin-Flip" onPress={playGame} />
SECURITY & FAIRNESS
The randomness uses Python’s random.random() which is fine for a low-stakes promo game.
For main-net launches swap in secrets.SystemRandom().random() or Chainlink VRF.  
The daily limit is enforced server-side (DB unique index) – users can’t bypass it even if they modify the app.  
Gasless: the 5 AHG are minted / transferred by the backend (same wallet that does the daily claim) so the user pays 0 gas for the game.
DONE
Restart your Flask server, rebuild the mobile bundle, and users will see a “Play Daily Coin-Flip” button right next to the “Claim 10 AHG” button.

Next
🎮 What’s Next?
Below is a road‑map that takes the “daily‑coin‑flip” game we just built and turns it into a fully‑featured, production‑ready mini‑app.
I’ll walk you through:

Leaderboard & Streaks  
More Game Variants (guess‑the‑number, spin‑wheel, etc.)  
Smart‑Contract‑Backed Game (Chainlink VRF)  
Frontend Enhancements (React‑Native UI, animations)  
Testing & CI  
Deployment (Docker + Fly.io / Render)
Feel free to skip any section you already have or want to focus on.  

1️⃣ Leaderboard & Streaks
1.1 Database Schema
-- Add a table for daily streaks
CREATE TABLE streaks (
    id          SERIAL PRIMARY KEY,
    user_id     INTEGER REFERENCES users(id) ON DELETE CASCADE,
    start_date  DATE NOT NULL,
    end_date    DATE,
    current_streak INTEGER NOT NULL DEFAULT 0,
    max_streak  INTEGER NOT NULL DEFAULT 0,
    UNIQUE(user_id, start_date)
);

-- Add a table for leaderboard (cached for performance)
CREATE TABLE leaderboard (
    id          SERIAL PRIMARY KEY,
    user_id     INTEGER REFERENCES users(id) ON DELETE CASCADE,
    total_wins  INTEGER NOT NULL DEFAULT 0,
    total_played INTEGER NOT NULL DEFAULT 0,
    last_updated TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
1.2 Backend Logic (Flask)
# in app.py or a new blueprint
from flask import Blueprint, jsonify
from models import db, User, GamePlay, Streak, Leaderboard
from sqlalchemy import func, desc

leaderboard_bp = Blueprint('leaderboard', __name__, url_prefix='/api/leaderboard')

@leaderboard_bp.route('/top', methods=['GET'])
def top_players():
    """Return top 10 players by total wins."""
    top = Leaderboard.query.order_by(desc(Leaderboard.total_wins)).limit(10).all()
    return jsonify([{
        'user': u.id,
        'wins': l.total_wins,
        'played': l.total_played
    } for u, l in [(User.query.get(l.user_id), l) for l in top]])

@leaderboard_bp.route('/streak', methods=['GET'])
def user_streak():
    """Return current streak for the logged‑in user."""
    wallet = request.args.get('wallet_address')
    user = User.query.filter_by(wallet_address=wallet).first()
    if not user:
        return jsonify({'error': 'User not found'}), 404

    streak = Streak.query.filter_by(user_id=user.id).order_by(desc(Streak.start_date)).first()
    if not streak:
        return jsonify({'current': 0, 'max': 0})
    return jsonify({
        'current': streak.current_streak,
        'max': streak.max_streak
    })
1.3 Updating Streaks & Leaderboard on Game Play
def update_game_stats(user, won):
    # Update leaderboard
    lb = Leaderboard.query.filter_by(user_id=user.id).first()
    if not lb:
        lb = Leaderboard(user_id=user.id)
        db.session.add(lb)
    lb.total_played += 1
    if won:
        lb.total_wins += 1

    # Update streak
    today = date.today()
    streak = Streak.query.filter_by(user_id=user.id).order_by(desc(Streak.start_date)).first()
    if streak and streak.end_date == today - timedelta(days=1):
        streak.current_streak += 1
    else:
        streak = Streak(user_id=user.id, start_date=today, current_streak=1)
        db.session.add(streak)

    if streak.current_streak > streak.max_streak:
        streak.max_streak = streak.current_streak

    db.session.commit()
Call update_game_stats(user, won) right after the game logic in the /api/game/play endpoint.

2️⃣ More Game Variants
Below are two simple, gas‑less game ideas that you can add with minimal code.

2.1 Guess‑the‑Number (1‑10)
# Add a new endpoint
@game_bp.route('/guess', methods=['POST'])
def guess_number():
    data = request.json
    wallet = data.get('wallet_address')
    guess = int(data.get('guess'))

    user = User.query.filter_by(wallet_address=wallet).first()
    if not user or not user.world_id_verified:
        return jsonify({'error': 'User not verified'}), 403

    today = date.today()
    if GamePlay.query.filter(GamePlay.user_id==user.id,
                              func.date(GamePlay.played_at)==today).first():
        return jsonify({'error': 'Already played today'}), 400

    correct = random.randint(1, 10)
    won = (guess == correct)

    play = GamePlay(user_id=user.id, won=won)
    db.session.add(play)
    if won:
        stake = Staking(user_id=user.id, amount=10)  # 10 AHG for correct guess
        db.session.add(stake)

    db.session.commit()
    update_game_stats(user, won)

    return jsonify({
        'won': won,
        'correct': correct,
        'message': f'You guessed {"right" if won else "wrong"}! The correct number was {correct}.'
    })
2.2 Spin‑the‑Wheel (10‑slot)
# Add a new endpoint
@game_bp.route('/spin', methods=['POST'])
def spin_wheel():
    data = request.json
    wallet = data.get('wallet_address')
    user = User.query.filter_by(wallet_address=wallet).first()
    if not user or not user.world_id_verified:
        return jsonify({'error': 'User not verified'}), 403

    today = date.today()
    if GamePlay.query.filter(GamePlay.user_id==user.id,
                              func.date(GamePlay.played_at)==today).first():
        return jsonify({'error': 'Already played today'}), 400

    # 10 slots: 0-9.  Slots 0-2 give 10 AHG, others give nothing.
    slot = random.randint(0, 9)
    won = slot < 3
    play = GamePlay(user_id=user.id, won=won)
    db.session.add(play)
    if won:
        stake = Staking(user_id=user.id, amount=10)
        db.session.add(stake)

    db.session.commit()
    update_game_stats(user, won)

    return jsonify({
        'won': won,
        'slot': slot,
        'message': f'You landed on slot {slot}! {("You win 10 AHG!" if won else "Better luck next time.")}'
    })
3️⃣ Smart‑Contract‑Backed Game (Optional)
If you want provably fair results, use Chainlink VRF or a simple on‑chain oracle.
A minimal approach:

Deploy a GameContract that exposes play() which returns a random number (via Chainlink VRF).  
The backend calls GameContract.play() and receives the outcome.  
The backend then mints / stakes tokens accordingly.
Why?  

Users trust that the result is not manipulated.  
You can later add more complex games (lottery, slot machine) that rely on on‑chain randomness.
4️⃣ Frontend Enhancements (React‑Native)
4.1 UI Components
// GameScreen.jsx
import { useState } from 'react';
import { View, Text, Button, Alert, TextInput } from 'react-native';

export default function GameScreen({ wallet }) {
  const [guess, setGuess] = useState('');

  const playCoinFlip = async () => {
    const res = await fetch(`${API}/api/game/play`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ wallet_address: wallet })
    });
    const data = await res.json();
    Alert.alert('Result', data.message);
  };

  const playGuess = async () => {
    const res = await fetch(`${API}/api/game/guess`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ wallet_address: wallet, guess })
