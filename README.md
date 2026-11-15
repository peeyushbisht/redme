# FlashLock — Database Concurrency & OS Synchronization Lab

**Team CodeSense:** Ayush Pilangwal | Shivansh Bijalwan | Peeyush Bisht | Kunj Tyagi  
**Course:** Project-Based Learning (Operating Systems + Database Management Systems)

---

## Architecture Overview

FlashLock demonstrates how concurrent flash-sale traffic behaves across three genuine database concurrency control strategies in PostgreSQL, paired with a C POSIX worker-thread engine:

1. **Atomic Update (`AtomicStrategy`)**
   Executes a single conditional `UPDATE` statement (`UPDATE products SET stock = stock - :qty, version = version + 1 WHERE id = :id AND stock >= :qty`).
2. **Pessimistic Locking (`PessimisticStrategy`)**
   Acquires an exclusive database row lock (`SELECT ... FROM products WHERE id = :id FOR UPDATE`) inside a transaction boundary before checking and decrementing stock.
3. **Optimistic Locking (`OptimisticStrategy`)**
   Reads the current row `version` without blocking readers and verifies `WHERE id = :id AND version = :expectedVersion AND stock >= :qty` on write, explicitly detecting version collisions under concurrent contention.
4. **C POSIX Worker Engine (`engine/`)**
   Handles bounded circular queue buffering, mutexes, counting semaphores, and request scheduling (`FIFO`, `Priority`, `Priority + Aging`) over Unix Domain Sockets (`/tmp/flashlock.sock`).

---

## Setup & Execution

Requirements: PostgreSQL, gcc + make, JDK 17+, Maven, Python 3 (`pip install requests` for the stress test).

### 1. Database (clean state)
```bash
sudo service postgresql start
sudo -u postgres psql -c "ALTER USER postgres PASSWORD 'postgres'"   # once; matches application.properties
db/setup.sh        # creates the flashlock DB, schema, seed data, triggers, stored procedures, roles
```
Re-running `db/setup.sh` resets everything.

### 2. Compile & Start C Engine (Terminal 1)
```bash
cd engine
make clean && make
./bin/flashlock_engine
```

### 3. Start Spring Boot Backend (Terminal 2)
```bash
cd backend
mvn spring-boot:run
```

### 4. Serve Frontend Dashboard (Terminal 3)
```bash
cd frontend
python3 -m http.server 3000
```
Open `http://localhost:3000` in your browser.

### 5. CLI Concurrency Stress Test (Optional, Terminal 4)
```bash
python3 scripts/concurrency_test.py --strategy ATOMIC --requests 1000 --product 4 --reset-stock 100
python3 scripts/concurrency_test.py --strategy PESSIMISTIC --requests 1000 --product 4 --reset-stock 100
python3 scripts/concurrency_test.py --strategy OPTIMISTIC --requests 1000 --product 4 --reset-stock 100
```
Each run must end with `Overselling : NO (PASSED)` and a final stock of 0.

---

## DBMS Demos

Run everything with `db/run_demos.sh`, or one topic at a time. Errors printed by the demos are expected
(they show failures on purpose). The demos only touch scratch/temp tables or roll back their changes.

| Topic | Command |
|---|---|
| ACID + failure/rollback (atomicity, savepoint, constraints, durability) | `psql -d flashlock -f db/acid_demo.sql` |
| Isolation levels: READ COMMITTED, REPEATABLE READ, SERIALIZABLE (two live sessions) | `db/isolation_demo.sh` |
| Functional dependencies + normalization to 3NF | `psql -d flashlock -f db/normalization_3nf.sql` |
| Indexing + `EXPLAIN ANALYZE` on 100,000 rows, before/after | `psql -d flashlock -f db/indexing_benchmark.sql` |
| Deadlock (separate test case, not part of the purchase code) | `db/deadlock_demo.sh` |
| Triggers (stock audit, order normalisation) | `psql -d flashlock -f db/triggers.sql` |
| Stored procedures (`sp_place_order`, `sp_reset_product`) | `psql -d flashlock -f db/procedures.sql` |
| Users & roles (`flashlock_readonly`, `flashlock_app`, `report_user`, `app_user`) | `psql -d flashlock -f db/roles.sql` |

The shell demos connect with `PGHOST=localhost PGUSER=postgres PGPASSWORD=postgres`; override those variables if yours differ.
The schema (`products` 1—N `orders` via `product_id`) is already in 3NF, so no redesign or ER diagram was added.
