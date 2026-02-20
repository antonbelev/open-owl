# ADR-009: 2025 Year in Review Feature

**Status:** Proposed
**Date:** 2025-12-08
**Deciders:** Engineering Team, Product Owner, UX Designer
**Context:** Seasonal feature to celebrate users' Claude Code journey in 2025 with shareable stats

---

## Context and Problem Statement

As 2025 comes to a close, we have an opportunity to create a memorable, engaging experience for Open Owl users - a **Year in Review** feature inspired by Spotify Wrapped, GitHub Skyline, and similar end-of-year celebrations.

**Goals:**
1. **Delight Users**: Create a fun, visually stunning recap of their 2025 Claude Code usage
2. **Drive Engagement**: Give users a reason to explore the Metrics feature
3. **Viral Growth**: Enable easy sharing to help others discover Open Owl
4. **Seasonal Joy**: Add festive touches (snowflakes, winter theme) to celebrate the holidays

**Constraints:**
- **Time-Limited**: Feature visible Dec 15, 2025 - Jan 1, 2026 (auto-hides after)
- **Privacy-First**: All stats computed locally, nothing sent to servers
- **Non-Intrusive**: One-time notification popup, dismissable

---

## Feature Overview

### User Journey

```
┌─────────────────────────────────────────────────────────────────────┐
│                         USER JOURNEY                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. User opens Open Owl (first time after Dec 15)                │
│     ↓                                                               │
│  2. 🎄 Notification popup appears:                                  │
│     "Your 2025 Year in Review is ready! See your Claude journey"   │
│     [View Now] [Maybe Later]                                        │
│     ↓                                                               │
│  3. User clicks "View Now" OR navigates to Metrics → Year Review   │
│     ↓                                                               │
│  4. ❄️ Snowflakes overlay + festive UI loads                        │
│     ↓                                                               │
│  5. Animated story-style cards reveal stats one by one:            │
│     - Total tokens (with fun comparison)                           │
│     - Cost insights                                                │
│     - Most active day/month                                        │
│     - Favorite model                                               │
│     - Top projects                                                 │
│     - Fun achievements/badges                                      │
│     ↓                                                               │
│  6. Final card: Share your review!                                 │
│     [Share Image] [Copy Link] [Tweet]                              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Detailed Feature Design

### 1. Entry Points

#### 1.1 First-Time Notification (Modal Popup)

**Trigger Conditions:**
- Date is between Dec 15, 2025 00:00:00 and Jan 1, 2026 00:00:00
- User has not dismissed the notification before (stored in localStorage)
- User has at least 1 session from 2025 (worth showing a review)

**Design:**

```
┌──────────────────────────────────────────────────────────────────┐
│                                                           ✕      │
│                                                                  │
│                    🎄 ❄️ 🎅                                      │
│                                                                  │
│              Your 2025 Year in Review                            │
│                    is ready!                                     │
│                                                                  │
│     See how you used Claude Code this year -                     │
│     your tokens, projects, and AI adventures await!             │
│                                                                  │
│     ┌─────────────────┐  ┌─────────────────────┐                │
│     │   Maybe Later   │  │   ✨ View Now ✨    │                │
│     └─────────────────┘  └─────────────────────┘                │
│                                                                  │
│     □ Don't show this again                                      │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

**Technical Implementation:**

```typescript
// src/renderer/components/year-review/YearReviewNotification.tsx

interface NotificationState {
  hasSeenNotification: boolean;
  hasDismissedPermanently: boolean;
  lastShownAt: string | null;
}

export const YearReviewNotification: React.FC = () => {
  const [isVisible, setIsVisible] = useState(false);
  const [dontShowAgain, setDontShowAgain] = useState(false);
  const navigate = useNavigate();

  useEffect(() => {
    const checkShouldShow = async () => {
      // Check date range (Dec 15, 2025 - Jan 1, 2026)
      const now = new Date();
      const startDate = new Date('2025-12-15T00:00:00');
      const endDate = new Date('2026-01-01T00:00:00');

      if (now < startDate || now >= endDate) {
        return; // Outside the feature window
      }

      // Check localStorage for dismissal
      const state = localStorage.getItem('yearReview2025');
      if (state) {
        const parsed: NotificationState = JSON.parse(state);
        if (parsed.hasDismissedPermanently) return;
        if (parsed.hasSeenNotification) return;
      }

      // Check if user has 2025 data
      const metrics = await window.electronAPI.computeMetrics({
        filter: { days: 365 }
      });

      if (metrics.success && metrics.data?.sessions.length > 0) {
        // Has data, show notification
        setIsVisible(true);
      }
    };

    checkShouldShow();
  }, []);

  const handleViewNow = () => {
    localStorage.setItem('yearReview2025', JSON.stringify({
      hasSeenNotification: true,
      hasDismissedPermanently: dontShowAgain,
      lastShownAt: new Date().toISOString()
    }));
    setIsVisible(false);
    navigate('/metrics/year-review');
  };

  const handleMaybeLater = () => {
    localStorage.setItem('yearReview2025', JSON.stringify({
      hasSeenNotification: true,
      hasDismissedPermanently: dontShowAgain,
      lastShownAt: new Date().toISOString()
    }));
    setIsVisible(false);
  };

  if (!isVisible) return null;

  return (
    <Modal isOpen={isVisible} onClose={handleMaybeLater}>
      <div className="year-review-notification">
        <div className="festive-header">🎄 ❄️ 🎅</div>
        <h2>Your 2025 Year in Review is ready!</h2>
        <p>
          See how you used Claude Code this year -
          your tokens, projects, and AI adventures await!
        </p>
        <div className="actions">
          <button onClick={handleMaybeLater} className="btn-secondary">
            Maybe Later
          </button>
          <button onClick={handleViewNow} className="btn-primary sparkle">
            ✨ View Now ✨
          </button>
        </div>
        <label className="dont-show">
          <input
            type="checkbox"
            checked={dontShowAgain}
            onChange={(e) => setDontShowAgain(e.target.checked)}
          />
          Don't show this again
        </label>
      </div>
    </Modal>
  );
};
```

#### 1.2 Menu Entry Point

Add a temporary menu item under Metrics:

```
Sidebar:
├── Dashboard
├── Settings
├── Metrics
│   ├── Overview
│   ├── Sessions
│   └── 🎄 2025 Year in Review  ← NEW (with festive icon)
└── ...
```

**Visibility Logic:**
- Show menu item only between Dec 15, 2025 - Jan 1, 2026
- After Jan 1, 2026: Menu item disappears, route returns 404

```typescript
// src/renderer/components/layout/Sidebar.tsx

const isYearReviewPeriod = () => {
  const now = new Date();
  const start = new Date('2025-12-15T00:00:00');
  const end = new Date('2026-01-01T00:00:00');
  return now >= start && now < end;
};

// In render:
{isYearReviewPeriod() && (
  <NavLink to="/metrics/year-review" className="nav-item festive">
    <span className="icon">🎄</span>
    <span className="label">2025 Year in Review</span>
    <span className="badge-new">NEW</span>
  </NavLink>
)}
```

---

### 2. Year in Review Experience

#### 2.1 Visual Theme: Winter Wonderland

**Design Elements:**
- **Background**: Deep blue gradient (#0a1628 → #1a365d)
- **Snowflakes**: Animated CSS snowflakes overlay (non-intrusive)
- **Accent Colors**: Warm gold (#fbbf24), soft white (#f8fafc), ice blue (#93c5fd)
- **Typography**: Large, bold stats with subtle glow effects
- **Cards**: Frosted glass effect (backdrop-blur)

**Snowflakes Implementation:**

```typescript
// src/renderer/components/year-review/SnowflakesOverlay.tsx

export const SnowflakesOverlay: React.FC = () => {
  const snowflakes = useMemo(() => {
    return Array.from({ length: 50 }, (_, i) => ({
      id: i,
      left: `${Math.random() * 100}%`,
      animationDelay: `${Math.random() * 10}s`,
      animationDuration: `${10 + Math.random() * 20}s`,
      opacity: 0.3 + Math.random() * 0.5,
      size: 4 + Math.random() * 8,
    }));
  }, []);

  return (
    <div className="snowflakes-overlay" aria-hidden="true">
      {snowflakes.map((flake) => (
        <div
          key={flake.id}
          className="snowflake"
          style={{
            left: flake.left,
            animationDelay: flake.animationDelay,
            animationDuration: flake.animationDuration,
            opacity: flake.opacity,
            width: flake.size,
            height: flake.size,
          }}
        >
          ❄
        </div>
      ))}
    </div>
  );
};
```

**CSS Animation:**

```css
/* src/renderer/styles/year-review.css */

.snowflakes-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  pointer-events: none;
  z-index: 100;
  overflow: hidden;
}

.snowflake {
  position: absolute;
  top: -20px;
  color: white;
  font-size: inherit;
  animation: fall linear infinite;
}

@keyframes fall {
  0% {
    transform: translateY(-10px) rotate(0deg);
  }
  100% {
    transform: translateY(100vh) rotate(360deg);
  }
}

/* Reduce motion for accessibility */
@media (prefers-reduced-motion: reduce) {
  .snowflake {
    animation: none;
    opacity: 0.3;
    top: 50%;
  }
}
```

#### 2.2 Story-Style Card Progression

The Year in Review is presented as a series of animated cards, similar to Instagram Stories or Spotify Wrapped. Users tap/click to advance through their personalized stats.

**Card Flow:**

```
[Card 1: Welcome]
    ↓
[Card 2: Total Tokens - The Big Number]
    ↓
[Card 3: Cost Summary - What You Spent]
    ↓
[Card 4: Most Active Period - Your Peak]
    ↓
[Card 5: Model Loyalty - Your AI Partner]
    ↓
[Card 6: Top Projects - Where You Coded]
    ↓
[Card 7: Achievements & Badges]
    ↓
[Card 8: Fun Facts & Comparisons]
    ↓
[Card 9: Share Your Review!]
```

**Card Component:**

```typescript
// src/renderer/components/year-review/ReviewCard.tsx

interface ReviewCardProps {
  children: React.ReactNode;
  onNext: () => void;
  onPrevious: () => void;
  currentIndex: number;
  totalCards: number;
  accentColor?: string;
}

export const ReviewCard: React.FC<ReviewCardProps> = ({
  children,
  onNext,
  onPrevious,
  currentIndex,
  totalCards,
  accentColor = '#fbbf24',
}) => {
  return (
    <motion.div
      className="review-card"
      initial={{ opacity: 0, scale: 0.9 }}
      animate={{ opacity: 1, scale: 1 }}
      exit={{ opacity: 0, scale: 0.9 }}
      transition={{ duration: 0.3 }}
      style={{ '--accent-color': accentColor } as React.CSSProperties}
    >
      {/* Progress dots */}
      <div className="progress-dots">
        {Array.from({ length: totalCards }).map((_, i) => (
          <div
            key={i}
            className={`dot ${i === currentIndex ? 'active' : ''} ${i < currentIndex ? 'completed' : ''}`}
          />
        ))}
      </div>

      {/* Card content */}
      <div className="card-content">{children}</div>

      {/* Navigation */}
      <div className="card-navigation">
        <button
          onClick={onPrevious}
          disabled={currentIndex === 0}
          className="nav-btn prev"
        >
          ← Back
        </button>
        <button onClick={onNext} className="nav-btn next">
          {currentIndex === totalCards - 1 ? 'Finish' : 'Next →'}
        </button>
      </div>
    </motion.div>
  );
};
```

---

### 3. Metrics & Stats to Display

Based on the available `MetricsData` structure, here are the stats we can compute and display:

#### 3.1 Card 2: Total Tokens - "The Big Number"

**Display:**
```
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│                    In 2025, you generated                      │
│                                                                │
│                    ✨ 12,847,293 ✨                            │
│                         tokens                                 │
│                                                                │
│        That's like reading War & Peace... 21 times! 📚        │
│                                                                │
│         Input: 8.2M  •  Output: 4.6M  •  Cached: 2.1M         │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

**Fun Comparisons (based on token count):**

```typescript
const getFunComparison = (totalTokens: number): string => {
  const warAndPeace = 580_000; // ~580K words ≈ tokens
  const harryPotterSeries = 1_084_000; // All 7 books
  const bible = 783_000;
  const tweets = 280; // Max tweet length

  if (totalTokens > 10_000_000) {
    const warPeaceCount = Math.round(totalTokens / warAndPeace);
    return `That's like reading War & Peace... ${warPeaceCount} times! 📚`;
  } else if (totalTokens > 5_000_000) {
    const hpCount = Math.round(totalTokens / harryPotterSeries);
    return `Enough to read the entire Harry Potter series ${hpCount}x! ⚡`;
  } else if (totalTokens > 1_000_000) {
    const bibleCount = Math.round(totalTokens / bible);
    return `That's ${bibleCount}x the length of the Bible! 📖`;
  } else if (totalTokens > 100_000) {
    const tweetCount = Math.round(totalTokens / tweets);
    return `You could have written ${tweetCount.toLocaleString()} tweets! 🐦`;
  } else {
    return `A solid start to your AI journey! 🚀`;
  }
};
```

#### 3.2 Card 3: Cost Summary - "What You Invested"

**Display:**
```
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│                   Your 2025 AI Investment                      │
│                                                                │
│                       💰 $127.45 💰                            │
│                                                                │
│              That's about $0.35/day on average!                │
│                                                                │
│     🟢 You saved $48.20 with prompt caching!                  │
│        (Cache efficiency: 67%)                                 │
│                                                                │
│                    "Money well spent!" 💪                      │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

**Fun Cost Comparisons:**

```typescript
const getCostComparison = (totalCost: number): string => {
  const coffees = totalCost / 5; // Avg latte price
  const netflixMonths = totalCost / 15.49;
  const avocadoToasts = totalCost / 12;

  if (totalCost > 500) {
    return `That's ${Math.round(coffees)} lattes ☕ - but way more productive!`;
  } else if (totalCost > 100) {
    return `About ${Math.round(netflixMonths)} months of Netflix 🎬`;
  } else if (totalCost > 20) {
    return `Or ${Math.round(avocadoToasts)} avocado toasts 🥑`;
  } else {
    return `Less than a fancy dinner! 🍽️`;
  }
};
```

#### 3.3 Card 4: Most Active Period - "Your Peak"

**Display:**
```
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│                   Your Peak Coding Month                       │
│                                                                │
│                     🔥 November 🔥                             │
│                                                                │
│              127 sessions • 2.4M tokens • $32.50               │
│                                                                │
│                 ┌────────────────────────┐                     │
│                 │  Monthly Activity Bar  │                     │
│                 │  J F M A M J J A S O N D                     │
│                 │  ▂ ▃ ▂ ▄ ▅ ▃ ▂ ▄ ▅ ▆ █ ▇                    │
│                 └────────────────────────┘                     │
│                                                                │
│           Busiest Day: Tuesday, Nov 12 (42 sessions!)         │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

**Implementation:**

```typescript
interface ActivityStats {
  peakMonth: {
    name: string;
    sessions: number;
    tokens: number;
    cost: number;
  };
  peakDay: {
    date: string;
    dayOfWeek: string;
    sessions: number;
  };
  monthlyData: { month: string; value: number }[];
}

const computeActivityStats = (daily: DailyStats[]): ActivityStats => {
  // Group by month
  const monthlyMap = new Map<string, { sessions: number; tokens: number; cost: number }>();

  for (const day of daily) {
    const month = day.date.slice(0, 7); // YYYY-MM
    const existing = monthlyMap.get(month) || { sessions: 0, tokens: 0, cost: 0 };
    monthlyMap.set(month, {
      sessions: existing.sessions + day.sessionCount,
      tokens: existing.tokens + day.totalTokens,
      cost: existing.cost + day.cost,
    });
  }

  // Find peak month
  let peakMonth = { name: '', sessions: 0, tokens: 0, cost: 0 };
  for (const [month, data] of monthlyMap) {
    if (data.sessions > peakMonth.sessions) {
      peakMonth = {
        name: new Date(month + '-01').toLocaleString('default', { month: 'long' }),
        ...data,
      };
    }
  }

  // Find peak day
  const sortedDays = [...daily].sort((a, b) => b.sessionCount - a.sessionCount);
  const topDay = sortedDays[0];

  return {
    peakMonth,
    peakDay: {
      date: topDay.date,
      dayOfWeek: new Date(topDay.date).toLocaleString('default', { weekday: 'long' }),
      sessions: topDay.sessionCount,
    },
    monthlyData: Array.from(monthlyMap.entries()).map(([month, data]) => ({
      month,
      value: data.sessions,
    })),
  };
};
```

#### 3.4 Card 5: Model Loyalty - "Your AI Partner"

**Display:**
```
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│                    Your Favorite Claude                        │
│                                                                │
│                      🤖 Sonnet 4 🤖                            │
│                                                                │
│                   "The Balanced Genius"                        │
│                                                                │
│                   847 sessions (72%)                           │
│                                                                │
│                    ┌─────────────────┐                         │
│                    │   🟣 72% Sonnet │                         │
│                    │   🟠 21% Opus   │                         │
│                    │   🔵  7% Haiku  │                         │
│                    └─────────────────┘                         │
│                                                                │
│        "You appreciate quality AND speed. Smart choice!"      │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

**Model Personality Descriptions:**

```typescript
const MODEL_PERSONALITIES: Record<string, { title: string; description: string; emoji: string }> = {
  'claude-sonnet': {
    title: 'The Balanced Genius',
    description: 'You appreciate quality AND speed. Smart choice!',
    emoji: '🎯',
  },
  'claude-opus': {
    title: 'The Perfectionist',
    description: 'You demand the best. Quality over everything!',
    emoji: '👑',
  },
  'claude-haiku': {
    title: 'The Speed Demon',
    description: 'Fast and efficient - you value your time!',
    emoji: '⚡',
  },
};

const getModelPersonality = (modelName: string) => {
  if (modelName.includes('sonnet')) return MODEL_PERSONALITIES['claude-sonnet'];
  if (modelName.includes('opus')) return MODEL_PERSONALITIES['claude-opus'];
  if (modelName.includes('haiku')) return MODEL_PERSONALITIES['claude-haiku'];
  return { title: 'The Explorer', description: 'Always trying new things!', emoji: '🧭' };
};
```

#### 3.5 Card 6: Top Projects - "Where You Coded"

**Display:**
```
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│                     Your Top Projects                          │
│                                                                │
│    🥇 open-owl                    312 sessions • $45.20      │
│       "Your passion project!"                                  │
│                                                                │
│    🥈 acme-api                      187 sessions • $28.90      │
│       "The workhorse"                                          │
│                                                                │
│    🥉 personal-website               89 sessions • $12.40     │
│       "Side hustle energy"                                     │
│                                                                │
│              + 12 other projects                               │
│                                                                │
│       You worked on 15 different projects this year! 🎨       │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

#### 3.6 Card 7: Achievements & Badges

**Display:**
```
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│                    🏆 Your 2025 Badges 🏆                      │
│                                                                │
│     ┌──────┐    ┌──────┐    ┌──────┐    ┌──────┐             │
│     │  🌅  │    │  🦉  │    │  💎  │    │  🔥  │             │
│     │Early │    │Night │    │Cache │    │ Hot  │             │
│     │ Bird │    │ Owl  │    │Master│    │Streak│             │
│     └──────┘    └──────┘    └──────┘    └──────┘             │
│                                                                │
│     Early Bird: 47 sessions before 8am ☀️                     │
│     Night Owl: 89 sessions after 10pm 🌙                       │
│     Cache Master: 67% cache efficiency 💾                      │
│     Hot Streak: 23-day coding streak in October! 📅           │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

**Badge Definitions:**

```typescript
interface Badge {
  id: string;
  name: string;
  emoji: string;
  description: string;
  condition: (data: MetricsData) => boolean;
  detail: (data: MetricsData) => string;
}

const BADGES: Badge[] = [
  {
    id: 'early-bird',
    name: 'Early Bird',
    emoji: '🌅',
    description: '20+ sessions before 8am',
    condition: (data) => {
      const earlyCount = data.sessions.filter((s) => {
        const hour = new Date(s.startTime).getHours();
        return hour < 8;
      }).length;
      return earlyCount >= 20;
    },
    detail: (data) => {
      const count = data.sessions.filter((s) => new Date(s.startTime).getHours() < 8).length;
      return `${count} sessions before 8am ☀️`;
    },
  },
  {
    id: 'night-owl',
    name: 'Night Owl',
    emoji: '🦉',
    description: '20+ sessions after 10pm',
    condition: (data) => {
      const lateCount = data.sessions.filter((s) => {
        const hour = new Date(s.startTime).getHours();
        return hour >= 22;
      }).length;
      return lateCount >= 20;
    },
    detail: (data) => {
      const count = data.sessions.filter((s) => new Date(s.startTime).getHours() >= 22).length;
      return `${count} sessions after 10pm 🌙`;
    },
  },
  {
    id: 'cache-master',
    name: 'Cache Master',
    emoji: '💎',
    description: '50%+ cache efficiency',
    condition: (data) => data.summary.cacheEfficiency >= 50,
    detail: (data) => `${Math.round(data.summary.cacheEfficiency)}% cache efficiency 💾`,
  },
  {
    id: 'hot-streak',
    name: 'Hot Streak',
    emoji: '🔥',
    description: '10+ consecutive coding days',
    condition: (data) => {
      const streak = computeLongestStreak(data.daily);
      return streak >= 10;
    },
    detail: (data) => {
      const streak = computeLongestStreak(data.daily);
      return `${streak}-day coding streak! 📅`;
    },
  },
  {
    id: 'million-club',
    name: 'Million Club',
    emoji: '🎰',
    description: '1M+ tokens generated',
    condition: (data) => data.summary.totalTokens >= 1_000_000,
    detail: (data) => `${(data.summary.totalTokens / 1_000_000).toFixed(1)}M tokens! 🚀`,
  },
  {
    id: 'project-hopper',
    name: 'Project Hopper',
    emoji: '🐰',
    description: '10+ different projects',
    condition: (data) => data.byProject.length >= 10,
    detail: (data) => `${data.byProject.length} different projects! 🎨`,
  },
  {
    id: 'opus-lover',
    name: 'Opus Lover',
    emoji: '👑',
    description: '100+ Opus sessions',
    condition: (data) => {
      const opusStats = data.byModel.find((m) => m.model.includes('opus'));
      return (opusStats?.sessionCount || 0) >= 100;
    },
    detail: (data) => {
      const opusStats = data.byModel.find((m) => m.model.includes('opus'));
      return `${opusStats?.sessionCount || 0} Opus sessions! 👑`;
    },
  },
  {
    id: 'weekend-warrior',
    name: 'Weekend Warrior',
    emoji: '⚔️',
    description: '50+ weekend sessions',
    condition: (data) => {
      const weekendCount = data.sessions.filter((s) => {
        const day = new Date(s.startTime).getDay();
        return day === 0 || day === 6;
      }).length;
      return weekendCount >= 50;
    },
    detail: (data) => {
      const count = data.sessions.filter((s) => {
        const day = new Date(s.startTime).getDay();
        return day === 0 || day === 6;
      }).length;
      return `${count} weekend sessions! 🎮`;
    },
  },
];

const computeLongestStreak = (daily: DailyStats[]): number => {
  if (daily.length === 0) return 0;

  const sortedDays = [...daily].sort(
    (a, b) => new Date(a.date).getTime() - new Date(b.date).getTime()
  );

  let maxStreak = 1;
  let currentStreak = 1;

  for (let i = 1; i < sortedDays.length; i++) {
    const prevDate = new Date(sortedDays[i - 1].date);
    const currDate = new Date(sortedDays[i].date);
    const diffDays = (currDate.getTime() - prevDate.getTime()) / (1000 * 60 * 60 * 24);

    if (diffDays === 1) {
      currentStreak++;
      maxStreak = Math.max(maxStreak, currentStreak);
    } else {
      currentStreak = 1;
    }
  }

  return maxStreak;
};
```

#### 3.7 Card 8: Fun Facts & Comparisons

**Display:**
```
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│                      Fun Facts About You                       │
│                                                                │
│     📊 You're in the top 10% of Claude Code users!            │
│        (based on session count)                                │
│                                                                │
│     ⏱️ Average session: 12 minutes                            │
│        Your longest: 2h 34m on Sep 15                          │
│                                                                │
│     💬 You exchanged 4,847 messages with Claude                │
│        That's like 16 messages per day!                        │
│                                                                │
│     🔄 You ran 1,247 tool calls                                │
│        (File edits, bash commands, searches...)                │
│                                                                │
│     🌍 You coded in 3 different timezones!                     │
│        (PST, EST, UTC)                                         │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

---

### 4. Social Sharing Feature

The final card enables users to share their Year in Review, helping spread awareness of Open Owl.

#### 4.1 Share Card Design

**Display:**
```
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│                   Share Your 2025 Journey!                     │
│                                                                │
│     ┌──────────────────────────────────────────────────┐      │
│     │                                                  │      │
│     │     🎄 My 2025 Claude Code Year in Review 🎄    │      │
│     │                                                  │      │
│     │     ✨ 12.8M tokens generated                   │      │
│     │     💰 $127 invested in AI                      │      │
│     │     🏆 347 coding sessions                      │      │
│     │     🔥 23-day hot streak                        │      │
│     │                                                  │      │
│     │     Powered by Open Owl                       │      │
│     │     github.com/antonbelev/open-owl            │      │
│     │                                                  │      │
│     └──────────────────────────────────────────────────┘      │
│                     ↑ Preview Card ↑                          │
│                                                                │
│     ┌──────────┐  ┌──────────┐  ┌──────────┐                 │
│     │📱 Save   │  │📋 Copy   │  │🐦 Tweet  │                 │
│     │  Image   │  │  Stats   │  │   It!    │                 │
│     └──────────┘  └──────────┘  └──────────┘                 │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

#### 4.2 Share Image Generation

Generate a beautiful PNG image that users can save and share on social media.

**Implementation:**

```typescript
// src/renderer/components/year-review/ShareImageGenerator.tsx

import html2canvas from 'html2canvas';

interface ShareStats {
  totalTokens: number;
  totalCost: number;
  totalSessions: number;
  longestStreak: number;
  topModel: string;
  topProject: string;
}

export const generateShareImage = async (stats: ShareStats): Promise<Blob> => {
  // Create hidden DOM element with share card design
  const shareCard = document.createElement('div');
  shareCard.id = 'share-card-temp';
  shareCard.innerHTML = `
    <div style="
      width: 600px;
      height: 800px;
      background: linear-gradient(135deg, #0a1628 0%, #1a365d 50%, #312e81 100%);
      padding: 40px;
      font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
      color: white;
      display: flex;
      flex-direction: column;
      align-items: center;
      justify-content: space-between;
    ">
      <div style="text-align: center;">
        <div style="font-size: 48px; margin-bottom: 20px;">🎄 ❄️ 🎅</div>
        <h1 style="font-size: 28px; margin: 0; color: #fbbf24;">
          My 2025 Claude Code
        </h1>
        <h2 style="font-size: 24px; margin: 10px 0; font-weight: normal;">
          Year in Review
        </h2>
      </div>

      <div style="
        background: rgba(255,255,255,0.1);
        border-radius: 20px;
        padding: 30px 40px;
        backdrop-filter: blur(10px);
        width: 100%;
      ">
        <div style="display: flex; justify-content: space-between; margin-bottom: 25px;">
          <div style="text-align: center;">
            <div style="font-size: 36px; font-weight: bold; color: #fbbf24;">
              ${formatNumber(stats.totalTokens)}
            </div>
            <div style="font-size: 14px; opacity: 0.8;">tokens</div>
          </div>
          <div style="text-align: center;">
            <div style="font-size: 36px; font-weight: bold; color: #34d399;">
              $${stats.totalCost.toFixed(0)}
            </div>
            <div style="font-size: 14px; opacity: 0.8;">invested</div>
          </div>
        </div>

        <div style="display: flex; justify-content: space-between;">
          <div style="text-align: center;">
            <div style="font-size: 28px; font-weight: bold;">
              ${stats.totalSessions}
            </div>
            <div style="font-size: 14px; opacity: 0.8;">sessions</div>
          </div>
          <div style="text-align: center;">
            <div style="font-size: 28px; font-weight: bold; color: #f97316;">
              🔥 ${stats.longestStreak}
            </div>
            <div style="font-size: 14px; opacity: 0.8;">day streak</div>
          </div>
        </div>
      </div>

      <div style="
        display: flex;
        gap: 30px;
        margin: 20px 0;
      ">
        <div style="text-align: center;">
          <div style="font-size: 40px;">🤖</div>
          <div style="font-size: 12px;">Top Model</div>
          <div style="font-size: 14px; font-weight: bold;">${stats.topModel}</div>
        </div>
        <div style="text-align: center;">
          <div style="font-size: 40px;">📁</div>
          <div style="font-size: 12px;">Top Project</div>
          <div style="font-size: 14px; font-weight: bold;">${stats.topProject}</div>
        </div>
      </div>

      <div style="text-align: center; margin-top: auto;">
        <div style="
          font-size: 14px;
          opacity: 0.7;
          margin-bottom: 10px;
        ">
          Created with
        </div>
        <div style="
          display: flex;
          align-items: center;
          justify-content: center;
          gap: 8px;
        ">
          <span style="font-size: 24px;">🦉</span>
          <span style="font-size: 18px; font-weight: bold;">Open Owl</span>
        </div>
        <div style="
          font-size: 12px;
          opacity: 0.5;
          margin-top: 5px;
        ">
          github.com/antonbelev/open-owl
        </div>
      </div>
    </div>
  `;

  document.body.appendChild(shareCard);

  try {
    const canvas = await html2canvas(shareCard.firstElementChild as HTMLElement, {
      backgroundColor: null,
      scale: 2, // High resolution
    });

    return new Promise((resolve) => {
      canvas.toBlob((blob) => {
        resolve(blob!);
      }, 'image/png');
    });
  } finally {
    document.body.removeChild(shareCard);
  }
};

const formatNumber = (num: number): string => {
  if (num >= 1_000_000) return `${(num / 1_000_000).toFixed(1)}M`;
  if (num >= 1_000) return `${(num / 1_000).toFixed(1)}K`;
  return num.toString();
};
```

#### 4.3 Share Actions

```typescript
// src/renderer/components/year-review/ShareActions.tsx

export const ShareActions: React.FC<{ stats: ShareStats }> = ({ stats }) => {
  const handleSaveImage = async () => {
    const blob = await generateShareImage(stats);
    const url = URL.createObjectURL(blob);

    const link = document.createElement('a');
    link.href = url;
    link.download = 'open-owl-2025-year-in-review.png';
    link.click();

    URL.revokeObjectURL(url);

    // Track share event (local only)
    console.log('[YearReview] Image saved');
  };

  const handleCopyStats = () => {
    const text = `🎄 My 2025 Claude Code Year in Review 🎄

✨ ${formatNumber(stats.totalTokens)} tokens generated
💰 $${stats.totalCost.toFixed(0)} invested in AI
🏆 ${stats.totalSessions} coding sessions
🔥 ${stats.longestStreak}-day hot streak

Check out Open Owl: https://github.com/antonbelev/open-owl

#ClaudeCode #YearInReview #AI #Coding`;

    navigator.clipboard.writeText(text);

    // Show toast
    toast.success('Stats copied to clipboard!');
  };

  const handleTweet = () => {
    const text = encodeURIComponent(
      `🎄 My 2025 Claude Code Year in Review:

✨ ${formatNumber(stats.totalTokens)} tokens
💰 $${stats.totalCost.toFixed(0)} invested
🏆 ${stats.totalSessions} sessions
🔥 ${stats.longestStreak}-day streak

Check out @OpenOwl! github.com/antonbelev/open-owl

#ClaudeCode #YearInReview`
    );

    window.open(`https://twitter.com/intent/tweet?text=${text}`, '_blank');
  };

  return (
    <div className="share-actions">
      <button onClick={handleSaveImage} className="share-btn">
        <span className="icon">📱</span>
        <span className="label">Save Image</span>
      </button>

      <button onClick={handleCopyStats} className="share-btn">
        <span className="icon">📋</span>
        <span className="label">Copy Stats</span>
      </button>

      <button onClick={handleTweet} className="share-btn twitter">
        <span className="icon">🐦</span>
        <span className="label">Tweet It!</span>
      </button>
    </div>
  );
};
```

---

### 5. Technical Implementation

#### 5.1 New Files Structure

```
src/
├── renderer/
│   ├── pages/
│   │   └── YearReviewPage.tsx           # Main page component
│   ├── components/
│   │   └── year-review/
│   │       ├── index.ts                  # Barrel export
│   │       ├── YearReviewNotification.tsx # Popup notification
│   │       ├── SnowflakesOverlay.tsx     # Festive snow effect
│   │       ├── ReviewCard.tsx            # Story-style card wrapper
│   │       ├── cards/
│   │       │   ├── WelcomeCard.tsx       # Card 1
│   │       │   ├── TokensCard.tsx        # Card 2
│   │       │   ├── CostCard.tsx          # Card 3
│   │       │   ├── ActivityCard.tsx      # Card 4
│   │       │   ├── ModelCard.tsx         # Card 5
│   │       │   ├── ProjectsCard.tsx      # Card 6
│   │       │   ├── BadgesCard.tsx        # Card 7
│   │       │   ├── FunFactsCard.tsx      # Card 8
│   │       │   └── ShareCard.tsx         # Card 9
│   │       ├── ShareImageGenerator.tsx   # PNG generation
│   │       └── ShareActions.tsx          # Share buttons
│   ├── hooks/
│   │   └── useYearReview.ts              # Data fetching hook
│   └── styles/
│       └── year-review.css               # Festive styles
├── shared/
│   └── types/
│       └── year-review.types.ts          # TypeScript types
└── main/
    └── services/
        └── YearReviewService.ts          # 2025-specific data filtering
```

#### 5.2 Year Review Hook

```typescript
// src/renderer/hooks/useYearReview.ts

import { useQuery } from '@tanstack/react-query';
import type { MetricsData } from '@/shared/types';

export interface YearReviewData {
  metrics: MetricsData;
  badges: Badge[];
  activityStats: ActivityStats;
  shareStats: ShareStats;
  funFacts: FunFact[];
}

export const useYearReview = () => {
  return useQuery({
    queryKey: ['year-review-2025'],
    queryFn: async (): Promise<YearReviewData> => {
      // Fetch metrics for 2025 only
      const response = await window.electronAPI.computeMetrics({
        filter: { days: 365 }, // Full year
      });

      if (!response.success || !response.data) {
        throw new Error(response.error || 'Failed to load metrics');
      }

      // Filter to 2025 data only
      const data = filter2025Data(response.data);

      // Compute derived stats
      const badges = computeBadges(data);
      const activityStats = computeActivityStats(data.daily);
      const shareStats = computeShareStats(data);
      const funFacts = computeFunFacts(data);

      return {
        metrics: data,
        badges,
        activityStats,
        shareStats,
        funFacts,
      };
    },
    staleTime: 5 * 60 * 1000, // 5 minutes
    cacheTime: 30 * 60 * 1000, // 30 minutes
  });
};

const filter2025Data = (data: MetricsData): MetricsData => {
  const startOf2025 = new Date('2025-01-01T00:00:00');
  const endOf2025 = new Date('2025-12-31T23:59:59');

  return {
    ...data,
    sessions: data.sessions.filter((s) => {
      const date = new Date(s.startTime);
      return date >= startOf2025 && date <= endOf2025;
    }),
    daily: data.daily.filter((d) => {
      const date = new Date(d.date);
      return date >= startOf2025 && date <= endOf2025;
    }),
    // Recompute summary for 2025 data only
    summary: recomputeSummary(data.sessions, data.daily, startOf2025, endOf2025),
  };
};
```

#### 5.3 Main Page Component

```typescript
// src/renderer/pages/YearReviewPage.tsx

import { useState } from 'react';
import { AnimatePresence } from 'framer-motion';
import { useYearReview } from '@/renderer/hooks/useYearReview';
import {
  SnowflakesOverlay,
  ReviewCard,
  WelcomeCard,
  TokensCard,
  CostCard,
  ActivityCard,
  ModelCard,
  ProjectsCard,
  BadgesCard,
  FunFactsCard,
  ShareCard,
} from '@/renderer/components/year-review';

export const YearReviewPage: React.FC = () => {
  const { data, isLoading, error } = useYearReview();
  const [currentCard, setCurrentCard] = useState(0);

  // Check if feature should be visible
  const isFeatureActive = () => {
    const now = new Date();
    const start = new Date('2025-12-15T00:00:00');
    const end = new Date('2026-01-01T00:00:00');
    return now >= start && now < end;
  };

  if (!isFeatureActive()) {
    return (
      <div className="year-review-expired">
        <h2>The 2025 Year in Review has ended</h2>
        <p>Check back next year for your 2026 review!</p>
      </div>
    );
  }

  if (isLoading) {
    return (
      <div className="year-review-loading">
        <SnowflakesOverlay />
        <div className="loading-content">
          <div className="spinner" />
          <p>Preparing your 2025 journey...</p>
        </div>
      </div>
    );
  }

  if (error || !data) {
    return (
      <div className="year-review-error">
        <h2>Oops! Something went wrong</h2>
        <p>{error?.message || 'Failed to load your year in review'}</p>
      </div>
    );
  }

  // No data for 2025
  if (data.metrics.sessions.length === 0) {
    return (
      <div className="year-review-no-data">
        <SnowflakesOverlay />
        <h2>🎄 No 2025 Data Yet 🎄</h2>
        <p>Start using Claude Code to build your year in review!</p>
      </div>
    );
  }

  const cards = [
    <WelcomeCard key="welcome" />,
    <TokensCard key="tokens" data={data} />,
    <CostCard key="cost" data={data} />,
    <ActivityCard key="activity" data={data} />,
    <ModelCard key="model" data={data} />,
    <ProjectsCard key="projects" data={data} />,
    <BadgesCard key="badges" data={data} />,
    <FunFactsCard key="funfacts" data={data} />,
    <ShareCard key="share" data={data} />,
  ];

  const handleNext = () => {
    if (currentCard < cards.length - 1) {
      setCurrentCard((prev) => prev + 1);
    }
  };

  const handlePrevious = () => {
    if (currentCard > 0) {
      setCurrentCard((prev) => prev - 1);
    }
  };

  return (
    <div className="year-review-page">
      <SnowflakesOverlay />

      <div className="year-review-container">
        <AnimatePresence mode="wait">
          <ReviewCard
            key={currentCard}
            onNext={handleNext}
            onPrevious={handlePrevious}
            currentIndex={currentCard}
            totalCards={cards.length}
          >
            {cards[currentCard]}
          </ReviewCard>
        </AnimatePresence>
      </div>
    </div>
  );
};
```

#### 5.4 Feature Flag & Date Guard

```typescript
// src/shared/utils/yearReview.utils.ts

export const YEAR_REVIEW_2025 = {
  startDate: new Date('2025-12-15T00:00:00'),
  endDate: new Date('2026-01-01T00:00:00'),
  year: 2025,
};

export const isYearReviewActive = (): boolean => {
  const now = new Date();
  return now >= YEAR_REVIEW_2025.startDate && now < YEAR_REVIEW_2025.endDate;
};

export const isYearReviewNotificationDismissed = (): boolean => {
  const state = localStorage.getItem('yearReview2025');
  if (!state) return false;
  const parsed = JSON.parse(state);
  return parsed.hasDismissedPermanently || parsed.hasSeenNotification;
};

export const dismissYearReviewNotification = (permanent: boolean = false): void => {
  localStorage.setItem(
    'yearReview2025',
    JSON.stringify({
      hasSeenNotification: true,
      hasDismissedPermanently: permanent,
      lastShownAt: new Date().toISOString(),
    })
  );
};
```

---

### 6. Accessibility & Performance

#### 6.1 Accessibility Considerations

- **Reduced Motion**: Respect `prefers-reduced-motion` for snowflakes and card animations
- **Screen Readers**: All stats have proper ARIA labels
- **Keyboard Navigation**: Cards can be navigated with arrow keys
- **Color Contrast**: Ensure WCAG AA compliance for all text

```css
@media (prefers-reduced-motion: reduce) {
  .snowflake {
    animation: none;
  }

  .review-card {
    transition: none;
  }
}
```

#### 6.2 Performance Optimizations

- **Lazy Loading**: YearReviewPage loaded only when navigated to
- **Memoization**: Share image generation cached
- **Virtualization**: Large session lists use windowing
- **Code Splitting**: Year review components in separate chunk

```typescript
// src/renderer/routes.tsx
const YearReviewPage = lazy(() => import('./pages/YearReviewPage'));

// In router:
{
  path: '/metrics/year-review',
  element: (
    <Suspense fallback={<LoadingSpinner />}>
      <YearReviewPage />
    </Suspense>
  ),
}
```

---

### 7. Testing Strategy

#### 7.1 Unit Tests

```typescript
// tests/unit/components/year-review/badges.test.ts

describe('Badge Computation', () => {
  test('awards Early Bird badge for 20+ early sessions', () => {
    const data = createMockMetricsData({
      sessions: Array.from({ length: 25 }, (_, i) => ({
        ...mockSession,
        startTime: new Date(`2025-06-${i + 1}T06:00:00`), // 6am
      })),
    });

    const badges = computeBadges(data);

    expect(badges.find((b) => b.id === 'early-bird')).toBeDefined();
  });

  test('awards Cache Master badge for 50%+ cache efficiency', () => {
    const data = createMockMetricsData({
      summary: { ...mockSummary, cacheEfficiency: 67 },
    });

    const badges = computeBadges(data);

    expect(badges.find((b) => b.id === 'cache-master')).toBeDefined();
  });

  test('computes longest streak correctly', () => {
    const daily = [
      { date: '2025-10-01', ...mockDayStats },
      { date: '2025-10-02', ...mockDayStats },
      { date: '2025-10-03', ...mockDayStats },
      { date: '2025-10-05', ...mockDayStats }, // Gap
      { date: '2025-10-06', ...mockDayStats },
    ];

    expect(computeLongestStreak(daily)).toBe(3);
  });
});
```

#### 7.2 Integration Tests

```typescript
// tests/integration/year-review.test.ts

describe('Year Review Integration', () => {
  test('filters data to 2025 only', async () => {
    // Create mock data with 2024 and 2025 sessions
    const mockData = {
      sessions: [
        { ...mockSession, startTime: new Date('2024-11-15') },
        { ...mockSession, startTime: new Date('2025-03-20') },
        { ...mockSession, startTime: new Date('2025-09-10') },
      ],
    };

    vi.mocked(window.electronAPI.computeMetrics).mockResolvedValue({
      success: true,
      data: mockData,
    });

    render(<YearReviewPage />);

    await waitFor(() => {
      // Should only show 2 sessions (2025 data)
      expect(screen.getByText(/2 sessions/)).toBeInTheDocument();
    });
  });

  test('shows no-data message when user has no 2025 activity', async () => {
    vi.mocked(window.electronAPI.computeMetrics).mockResolvedValue({
      success: true,
      data: { sessions: [], daily: [], byModel: [], byProject: [], summary: mockSummary },
    });

    render(<YearReviewPage />);

    await waitFor(() => {
      expect(screen.getByText(/No 2025 Data Yet/)).toBeInTheDocument();
    });
  });
});
```

#### 7.3 E2E Tests

```typescript
// tests/e2e/year-review.spec.ts

test('Year in Review flow', async ({ page }) => {
  // Mock date to be within feature window
  await page.addInitScript(() => {
    Date.now = () => new Date('2025-12-20T12:00:00').getTime();
  });

  await page.goto('/');

  // Should see notification popup
  await expect(page.getByText('Your 2025 Year in Review is ready!')).toBeVisible();

  // Click View Now
  await page.getByRole('button', { name: /View Now/ }).click();

  // Should navigate to year review page
  await expect(page).toHaveURL('/metrics/year-review');

  // Should see snowflakes
  await expect(page.locator('.snowflakes-overlay')).toBeVisible();

  // Navigate through cards
  for (let i = 0; i < 8; i++) {
    await page.getByRole('button', { name: /Next/ }).click();
  }

  // Final card should have share options
  await expect(page.getByRole('button', { name: /Save Image/ })).toBeVisible();
  await expect(page.getByRole('button', { name: /Tweet/ })).toBeVisible();
});
```

---

### 8. Implementation Timeline

#### Phase 1: Core Infrastructure (Day 1-2)
- [ ] Create `year-review.types.ts` with all interfaces
- [ ] Create `yearReview.utils.ts` with date guards
- [ ] Create `useYearReview.ts` hook
- [ ] Add route and lazy loading setup

#### Phase 2: Visual Components (Day 3-4)
- [ ] Implement `SnowflakesOverlay.tsx`
- [ ] Implement `ReviewCard.tsx` with animations
- [ ] Create base CSS styles (`year-review.css`)

#### Phase 3: Stats Cards (Day 5-7)
- [ ] Implement all 9 card components
- [ ] Add fun comparisons and descriptions
- [ ] Implement badge computation logic

#### Phase 4: Social Sharing (Day 8)
- [ ] Implement `ShareImageGenerator.tsx`
- [ ] Add copy-to-clipboard functionality
- [ ] Add Twitter share integration

#### Phase 5: Notification & Entry Points (Day 9)
- [ ] Implement `YearReviewNotification.tsx`
- [ ] Add sidebar menu item with visibility logic
- [ ] Wire up notification dismissal storage

#### Phase 6: Testing & Polish (Day 10)
- [ ] Write unit tests for badge logic
- [ ] Write integration tests for data filtering
- [ ] Add E2E test for full flow
- [ ] Accessibility audit and fixes

---

### 9. Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Date logic bugs (show after Jan 1) | High | Comprehensive date tests, server-side backup date check |
| Poor performance with large datasets | Medium | Progressive loading, data sampling for very large datasets |
| Share image quality issues | Low | High-res canvas rendering (2x scale), test on multiple devices |
| Users without 2025 data see broken UI | Medium | Graceful empty state, minimum data threshold check |
| Accessibility issues with animations | Low | `prefers-reduced-motion` support, static fallbacks |

---

### 10. Success Metrics

**Engagement:**
- % of users who view Year in Review (target: 60%+)
- % of users who navigate through all cards (target: 40%+)
- Average time spent on Year Review (target: 2+ minutes)

**Virality:**
- Number of share image downloads
- Number of Tweet clicks
- GitHub stars/forks from referral traffic during feature period

**Technical:**
- Page load time < 2 seconds
- Zero crashes during feature period
- < 5 bug reports

---

### 11. Future Considerations (2026+)

- **Template System**: Make year-in-review generation generic for any year
- **Custom Themes**: Let users pick different visual themes (not just winter)
- **Comparison Mode**: Compare 2025 vs 2026 stats
- **Team Reviews**: Aggregate stats across team members (if team features added)
- **Export PDF**: Generate detailed PDF report for enterprise users

---

## Conclusion

The 2025 Year in Review feature will delight Open Owl users with a visually stunning, shareable summary of their AI coding journey. By combining the existing metrics infrastructure with engaging UX and social sharing, we can create a memorable experience that drives both engagement and organic growth.

**Key Differentiators:**
- Beautiful, festive design with snowflakes overlay
- Story-style card progression (like Spotify Wrapped)
- Fun comparisons and achievements/badges
- Easy social sharing with auto-generated images
- Privacy-first: All data stays local

**Recommendation:** Proceed with implementation, targeting release by Dec 15, 2025.

---

**Approval Signatures:**
- [ ] Product Owner: ___________________________
- [ ] Engineering Lead: ___________________________
- [ ] UX Designer: ___________________________

**Next Steps:**
1. Review and approve ADR
2. Create GitHub issue with implementation tasks
3. Begin Phase 1 development
4. Target Dec 15, 2025 for feature launch
