# Fantasy Soccer Optimizer - Design Document

## 1. Project Overview

A web-based tool that helps Fantasy Premier League players build optimal squads within budget constraints. The system analyzes player performance data, fixture difficulty, and team composition rules to recommend the highest-expected-value squad. This solves the combinatorial optimization problem that most fantasy players struggle with manually, while providing transparency into why each player was selected.

**Target Users:** Fantasy Premier League participants who want data-driven squad selection and competitive advantage.

## 2. System Architecture

### High-Level Components
```
┌─────────────┐         ┌──────────────────┐         ┌─────────────┐
│   React     │ ◄─────► │   Spring Boot    │ ◄─────► │ PostgreSQL  │
│  Frontend   │  HTTP   │   Backend API    │  JDBC   │  Database   │
└─────────────┘         └──────────────────┘         └─────────────┘
                               │      ▲
                               │      │
                               ▼      │
                        ┌─────────────┴───┐
                        │  Redis Cache    │
                        └─────────────────┘
                               ▲
                               │
                        ┌──────┴──────────┐
                        │  FPL API        │
                        │  (External)     │
                        └─────────────────┘
```

**Components:**
- **Frontend (React)**: User interface for inputting constraints and viewing optimized squads
- **Backend (Spring Boot)**: REST API handling optimization requests, data management, and business logic
- **Database (PostgreSQL)**: Persistent storage for player data, teams, fixtures, and historical stats
- **Cache (Redis)**: High-speed caching for optimization results and frequently accessed data
- **External API**: Fantasy Premier League official API for real-time player and fixture data

### Data Flow
1. Scheduled job fetches latest data from FPL API daily (6 AM)
2. Data is transformed and stored in PostgreSQL
3. User submits optimization request via React frontend
4. Backend checks Redis cache for similar request (based on constraint hash)
5. Cache miss: Optimization algorithm runs, result cached in Redis (TTL: 1 hour)
6. Cache hit: Result returned immediately
7. Response sent to frontend with optimized squad

## 3. Database Schema

### Player
```sql
CREATE TABLE player (
    id BIGINT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    team_id BIGINT REFERENCES team(id),
    position VARCHAR(3) NOT NULL, -- GK, DEF, MID, FWD
    price DECIMAL(4,1) NOT NULL,
    total_points INT NOT NULL,
    form DECIMAL(3,1),
    minutes_played INT,
    goals_scored INT,
    assists INT,
    clean_sheets INT,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_player_position ON player(position);
CREATE INDEX idx_player_team ON player(team_id);
CREATE INDEX idx_player_price ON player(price);
```

### Team
```sql
CREATE TABLE team (
    id BIGINT PRIMARY KEY,
    name VARCHAR(50) NOT NULL,
    short_name VARCHAR(3) NOT NULL,
    strength_overall INT,
    strength_attack INT,
    strength_defence INT,
    created_at TIMESTAMP DEFAULT NOW()
);
```

### Fixture
```sql
CREATE TABLE fixture (
    id BIGINT PRIMARY KEY,
    gameweek INT NOT NULL,
    home_team_id BIGINT REFERENCES team(id),
    away_team_id BIGINT REFERENCES team(id),
    home_difficulty INT, -- 1-5 scale
    away_difficulty INT,
    kickoff_time TIMESTAMP,
    finished BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_fixture_gameweek ON fixture(gameweek);
```

### GameweekStats
```sql
CREATE TABLE gameweek_stats (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    player_id BIGINT REFERENCES player(id),
    gameweek INT NOT NULL,
    points_scored INT,
    minutes_played INT,
    goals_scored INT,
    assists INT,
    bonus_points INT,
    price_at_time DECIMAL(4,1),
    created_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(player_id, gameweek)
);

CREATE INDEX idx_gameweek_stats_player ON gameweek_stats(player_id);
CREATE INDEX idx_gameweek_stats_week ON gameweek_stats(gameweek);
```

### OptimizationResult (Cache Table)
```sql
CREATE TABLE optimization_result (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    constraint_hash VARCHAR(64) UNIQUE NOT NULL,
    budget DECIMAL(5,1),
    formation VARCHAR(10),
    selected_player_ids JSON,
    total_cost DECIMAL(5,1),
    expected_points DECIMAL(6,1),
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_optimization_hash ON optimization_result(constraint_hash);
```

## 4. API Endpoints

### POST /api/optimize
Generates optimal squad based on constraints.

**Request:**
```json
{
  "budget": 100.0,
  "formation": "3-4-3",
  "requiredPlayerIds": [123, 456],
  "excludedPlayerIds": [789],
  "maxPlayersPerTeam": 3
}
```

**Response:**
```json
{
  "squad": [
    {
      "playerId": 123,
      "name": "Mohamed Salah",
      "team": "Liverpool",
      "position": "MID",
      "price": 13.0,
      "expectedPoints": 8.5,
      "reason": "High points-per-cost ratio, favorable fixtures"
    }
  ],
  "startingEleven": [123, 456, 789, ...],
  "bench": [111, 222, 333, 444],
  "totalCost": 99.5,
  "expectedPoints": 850,
  "formation": "3-4-3"
}
```

### GET /api/players
Retrieve filtered list of players.

**Query Parameters:**
- `position` (optional): GK, DEF, MID, FWD
- `maxPrice` (optional): decimal
- `minPoints` (optional): integer
- `teamId` (optional): integer

**Response:**
```json
{
  "players": [
    {
      "id": 123,
      "name": "Mohamed Salah",
      "team": "Liverpool",
      "position": "MID",
      "price": 13.0,
      "totalPoints": 185,
      "form": 7.2
    }
  ],
  "count": 50
}
```

### GET /api/fixtures/{gameweek}
Get fixtures for a specific gameweek.

**Response:**
```json
{
  "gameweek": 25,
  "fixtures": [
    {
      "id": 301,
      "homeTeam": "Liverpool",
      "awayTeam": "Manchester United",
      "homeDifficulty": 3,
      "awayDifficulty": 4,
      "kickoffTime": "2026-02-14T15:00:00Z"
    }
  ]
}
```

### POST /api/refresh-data
Manually trigger data sync from FPL API (admin endpoint).

**Response:**
```json
{
  "status": "success",
  "playersUpdated": 650,
  "fixturesUpdated": 38,
  "timestamp": "2026-01-28T21:45:00Z"
}
```

## 5. Backend Architecture (Spring Boot)

### Controllers
- **OptimizationController**: Handles `/api/optimize` endpoint
- **PlayerController**: Handles `/api/players` and player-related queries
- **FixtureController**: Handles `/api/fixtures/{gameweek}`
- **DataSyncController**: Handles `/api/refresh-data`

### Services
- **FPLDataService**: Fetches and syncs data from external FPL API
- **OptimizationService**: Core optimization algorithm implementation
- **PlayerAnalyticsService**: Calculates derived statistics (form, expected points)
- **CacheService**: Manages Redis caching logic

### Repositories
- **PlayerRepository**: JPA repository for Player entity
- **TeamRepository**: JPA repository for Team entity
- **FixtureRepository**: JPA repository for Fixture entity
- **GameweekStatsRepository**: JPA repository for GameweekStats entity

### Scheduled Tasks
- **Daily Data Refresh**: Runs at 6 AM daily, syncs latest FPL data
- **Cache Cleanup**: Runs hourly, removes stale optimization results

## 6. Optimization Algorithm

### MVP Approach: Greedy Algorithm

**Algorithm Steps:**
1. Calculate points-per-cost ratio for all players
2. Sort players by this ratio (descending)
3. Initialize empty squad with position counters (GK: 0/2, DEF: 0/5, MID: 0/5, FWD: 0/3)
4. For each player in sorted list:
   - Check if position quota not exceeded
   - Check if team quota not exceeded (max 3 per team)
   - Check if adding player stays within budget
   - If all constraints satisfied, add to squad
5. Continue until squad has 15 players
6. Select starting 11 based on formation (highest expected points)
7. Return squad

**Time Complexity:** O(n log n) where n = number of players (~650)

**Pros:**
- Fast execution (< 100ms)
- Simple to implement and debug
- Good enough for MVP

**Cons:**
- Not guaranteed to find optimal solution
- Doesn't consider player combinations or synergies

### Future Enhancement: Dynamic Programming

**Why DP would be better:**
- Guaranteed optimal solution
- Can handle complex constraints (chip usage, transfer planning)
- Better for multi-gameweek optimization

**Trade-offs:**
- Slower execution (~1-2 seconds)
- More complex implementation
- Higher memory usage

**Decision:** Start with greedy for MVP, benchmark against DP in V2.

### Input Constraints
```java
public class OptimizationRequest {
    private Double budget;              // e.g., 100.0
    private String formation;           // e.g., "3-4-3"
    private List<Long> requiredPlayerIds;  // Must include
    private List<Long> excludedPlayerIds;  // Must exclude
    private Integer maxPlayersPerTeam;  // Default: 3
}
```

### Output
```java
public class OptimizationResponse {
    private List<SelectedPlayer> squad;     // All 15 players
    private List<Long> startingEleven;      // Player IDs
    private List<Long> bench;               // Player IDs
    private Double totalCost;
    private Double expectedPoints;
    private String formation;
}
```

## 7. Caching Strategy

### Redis Layer

**Cache Keys:**
- `optimization:{hash}` - Optimization results keyed by constraint hash
- `players:all` - Complete player list (TTL: 5 minutes)
- `fpl:bootstrap` - Raw FPL API response (TTL: 1 hour)
- `fixtures:{gameweek}` - Fixtures for specific gameweek (TTL: 24 hours)

**Hash Generation:**
```java
String hash = MD5(budget + formation + requiredPlayers + excludedPlayers);
```

**TTL Strategy:**
- Optimization results: 1 hour (invalidate on new gameweek)
- Player lists: 5 minutes (frequently changing due to price updates)
- FPL API responses: 1 hour (reduce external API calls)
- Fixtures: 24 hours (rarely change mid-week)

**Cache Invalidation Triggers:**
- New gameweek starts → Clear all optimization caches
- Price changes detected → Clear player caches
- Manual refresh requested → Clear FPL API cache

## 8. Frontend (React)

### Pages
- **Home/Optimizer**: Main optimization interface
- **Player Browser**: Searchable/filterable player table
- **Squad Analyzer**: View and compare different squad compositions
- **About**: How the optimizer works, algorithm explanation

### Components
- **ConstraintForm**: Input fields for budget, formation, filters
- **PlayerFilter**: Position, price range, team selection
- **SquadDisplay**: Visual formation view with player cards
- **PlayerCard**: Individual player stats and info
- **ComparisonTable**: Side-by-side squad comparison

### State Management
- **MVP**: React hooks (useState, useEffect)
- **V2+**: Context API for global state (selected players, filters)
- **Future**: Consider Zustand if state becomes complex

## 9. DevOps & Deployment

### Docker Setup

**Backend Dockerfile:**
```dockerfile
FROM openjdk:17-jdk-slim
WORKDIR /app
COPY target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**Frontend Dockerfile:**
```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build
EXPOSE 3000
CMD ["npm", "start"]
```

**docker-compose.yml:**
```yaml
version: '3.8'
services:
  backend:
    build: ./backend
    ports:
      - "8080:8080"
    environment:
      - DATABASE_URL=jdbc:postgresql://db:5432/fantasy
      - REDIS_HOST=redis
    depends_on:
      - db
      - redis

  frontend:
    build: ./frontend
    ports:
      - "3000:3000"

  db:
    image: postgres:15
    environment:
      POSTGRES_DB: fantasy
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: password
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  postgres_data:
```

### CI/CD (GitHub Actions)

**Build and Test Pipeline:**
- Trigger on: Push to `main`, Pull Requests
- Steps: Checkout code → Build backend → Run tests → Build frontend
- Fail build if tests fail

**Deployment:**
- Platform options: Railway, Render, AWS Free Tier
- Auto-deploy on merge to `main`
- Environment variables managed via platform

## 10. Testing Strategy

### Unit Tests
- **OptimizationService**: Test algorithm with known inputs/outputs
- **PlayerAnalyticsService**: Verify calculations (form, expected points)
- **Utility classes**: Date helpers, constraint validators

**Example Test:**
```java
@Test
public void testGreedyOptimization_WithBudgetConstraint() {
    // Given: 100M budget, 3-4-3 formation
    // When: Run optimization
    // Then: Total cost <= 100M, formation respected
}
```

### Integration Tests
- **API Endpoints**: Test full request/response cycle
- **Database Operations**: Verify CRUD operations work correctly
- **Cache Behavior**: Ensure Redis caching works as expected

### Manual Testing Checklist
- [ ] Optimization produces valid squads (15 players, within budget)
- [ ] Formation constraints respected (correct positions)
- [ ] Team constraint respected (max 3 per team)
- [ ] Required players always included
- [ ] Excluded players never included
- [ ] Results make intuitive sense (high-value players selected)

## 11. Enhanced Features (V2+)

### Expected Points Model
Instead of using total points from last season, predict future performance:

**Factors:**
- **Fixture Difficulty**: Weight points by opponent strength (3 gameweeks ahead)
- **Form Trends**: Exponential moving average of last 5 gameweeks
- **Home/Away Splits**: Adjust based on venue
- **Minutes Prediction**: Factor in injury risk, rotation likelihood

**Formula:**
```
ExpectedPoints = BasePoints × FixtureMultiplier × FormMultiplier × MinutesProbability
```

### Multi-Gameweek Planning
Optimize not just current week, but next 3-5 gameweeks:
- Consider transfer costs (-4 points per transfer)
- Plan optimal wildcard/free hit timing
- Maximize cumulative points over period

### Risk Analysis
- **Variance Calculation**: Show confidence intervals for each player
- **Ceiling/Floor**: Best and worst-case scenarios
- **Ownership Consideration**: Flag differentials (low-owned, high-upside)

### Differential Analysis
Identify template-breaking picks:
- Compare against average Top 10k squad
- Highlight low-ownership players with favorable fixtures
- Calculate risk/reward for captaincy differentials

## 12. Tech Stack Justification

| Technology | Why Chosen |
|-----------|------------|
| **Spring Boot** | Industry standard for Java backends, excellent for RESTful APIs, strong ecosystem |
| **PostgreSQL** | Robust relational database, handles complex queries well, good for analytics |
| **Redis** | Blazing fast in-memory cache, perfect for expensive optimization results |
| **React** | Modern component-based UI, great developer experience, large community |
| **Docker** | Consistent environments across dev/prod, easy deployment, platform-agnostic |
| **JPA/Hibernate** | Simplifies database operations, ORM reduces boilerplate |
| **Lombok** | Reduces Java boilerplate (getters/setters/constructors) |

## 13. Open Questions / Decisions Needed

- [ ] **Algorithm choice**: Start with greedy or invest time in DP immediately?
  - *Leaning toward*: Greedy for MVP, DP for V2
  
- [ ] **Injured player handling**: Exclude automatically or show with warning?
  - *Leaning toward*: Flag in UI but allow selection with warning
  
- [ ] **Cache TTL**: Is 1 hour too long for optimization results?
  - *Need to test*: Monitor cache hit rate vs staleness
  
- [ ] **Frontend framework**: React or Next.js?
  - *Decision*: React for MVP (simpler), Next.js if SEO matters later
  
- [ ] **Authentication**: Do we need user accounts for saving squads?
  - *V1*: No auth, stateless
  - *V2*: Add auth for saved squads, history tracking

## 14. Development Phases

### Phase 1: MVP (Weeks 1-2)
- [ ] Set up Spring Boot project with PostgreSQL
- [ ] Implement FPL API data fetching
- [ ] Build basic greedy optimization algorithm
- [ ] Create REST API endpoints
- [ ] Simple React frontend (form + results display)
- [ ] Deploy to Railway/Render

### Phase 2: Polish (Week 3)
- [ ] Add Redis caching
- [ ] Improve frontend UI/UX
- [ ] Add player filtering and search
- [ ] Implement scheduled data refresh job
- [ ] Write unit tests for core logic
- [ ] Documentation (README, API docs)

### Phase 3: Advanced Features (Week 4+)
- [ ] Expected points model
- [ ] Multi-gameweek optimization
- [ ] Algorithm comparison (greedy vs DP)
- [ ] Risk analysis and variance
- [ ] Backtesting framework
- [ ] Performance optimization

## 15. Success Metrics

**Technical:**
- Optimization completes in < 500ms (greedy) or < 2s (DP)
- Cache hit rate > 40%
- API uptime > 99%
- Test coverage > 70%

**Product:**
- Generates valid squads 100% of the time
- Optimization results outperform random selection by 15%+
- User can go from landing page to optimized squad in < 60 seconds

---

**Last Updated:** January 28, 2026  
**Author:** Akshaj Sinha  
**Status:** Initial Design - Pre-Implementation
