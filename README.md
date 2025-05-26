# military_simmulation
# ⚔️ Military Simulation System Design (Strategic / Operational / Tactical)

This document outlines the design and initial implementation of a military simulation system leveraging H3 hexagons, Neo4j, Django, and DRF. The system simulates military movement and combat actions, integrated with real-time intelligence extraction.

---

## 🔧 Tech Stack Overview

| Layer        | Technology             | Purpose                                      |
|--------------|------------------------|----------------------------------------------|
| Backend      | Django + DRF           | API and simulation orchestration             |
| Geospatial   | H3 (Uber Hexagons)     | Battlefield grid (hexagon-based)             |
| Simulation DB| Neo4j                  | Graph simulation: units, power, connections  |
| Primary DB   | PostgreSQL             | Action logs, history, validation              |
| Frontend     | React (TypeScript)     | Command interface, live map, simulation input|
| Intelligence | LLM + Scrapers         | Parse and inject situational awareness       |

---

## 🧩 System Modules

### 1. 🛰️ Intelligence Collection
- Web scraping from social media (X, Telegram, blogs)
- Natural language processing to extract geolocated events
- Use of LLMs to detect threats and changes in battlefield state

### 2. 🗃️ Structured Data Ingestion
- Parses raw scraped data
- Extracts H3 hex ID, event type, actor units, estimates
- Pushes structured JSON to backend

### 3. 🧠 Backend Logic Layer
- Verifies and deduplicates intelligence
- Routes to PostgreSQL and Neo4j
- Triggers simulation updates if new threats/events arise

### 4. 🕸️ Neo4j Graph Database (Simulation Core)
- Hexagons as nodes (with H3 index)
- Units as nodes with:
  - `destructive_power_joule`
  - `kinetic_power_kw`
  - `defense_rating`
- Edges:
  - `:LOCATED_IN` between Unit → Hex
  - `:CONNECTED_TO` between Hex ↔ Hex

### 5. 🧪 Simulation Engine
- Receives simulation steps (e.g. move, engage)
- Applies outcomes, energy costs, deletions, updates
- Logs every step to PostgreSQL for history and rollback

---

## ⚙️ Simulation Step: MOVE

### Code Snippet

```python
def simulate_move(self, unit_id, from_h3, to_h3):
    with self.driver.session() as session:
        result = session.run(
            """
            MATCH (u:Unit {id: $unit_id})-[:LOCATED_IN]->(h:Hexagon {h3: $from_h3})
            RETURN u.kinetic_power_kw AS power_kw, u.destructive_power_joule AS power_j, u
            """,
            unit_id=unit_id,
            from_h3=from_h3
        )
        record = result.single()
        power_joule = record["power_j"]
        movement_cost = self.calculate_energy_cost(from_h3, to_h3)

        if power_joule < movement_cost:
            raise ValueError("Not enough energy to move")

        remaining_power = power_joule - movement_cost

        session.run(
            """
            MATCH (u:Unit {id: $unit_id})-[r:LOCATED_IN]->(h:Hexagon)
            DELETE r
            WITH u
            MATCH (target:Hexagon {h3: $to_h3})
            CREATE (u)-[:LOCATED_IN]->(target)
            SET u.destructive_power_joule = $remaining_power
            """,
            unit_id=unit_id,
            to_h3=to_h3,
            remaining_power=remaining_power
        )

        SimulationAction.objects.create(
            action_type="MOVE",
            unit_id=unit_id,
            from_hex=from_h3,
            to_hex=to_h3,
            cost_joules=movement_cost,
            remaining_power=remaining_power,
            timestamp=now()
        )


def simulate_engage(self, attacker_id, defender_id):
    with self.driver.session() as session:
        result = session.run(
            """
            MATCH (attacker:Unit {id: $attacker_id})-[:LOCATED_IN]->(hex:Hexagon)
            MATCH (defender:Unit {id: $defender_id})-[:LOCATED_IN]->(hex)
            RETURN attacker, attacker.destructive_power_joule AS atk_power,
                   defender, defender.destructive_power_joule AS def_power,
                   defender.defense_rating AS def_rating
            """,
            attacker_id=attacker_id,
            defender_id=defender_id
        )
        record = result.single()
        atk_power = record["atk_power"]
        def_power = record["def_power"]
        def_rating = record["def_rating"] or 1.0

        damage_dealt = int(atk_power * 0.7)
        net_damage = int(damage_dealt / def_rating)
        remaining_defender_power = def_power - net_damage
        remaining_attacker_power = int(atk_power * 0.3)

        if remaining_defender_power <= 0:
            session.run("MATCH (d:Unit {id: $defender_id}) DETACH DELETE d", defender_id=defender_id)
            outcome = "destroyed"
        else:
            session.run(
                "MATCH (d:Unit {id: $defender_id}) SET d.destructive_power_joule = $p",
                defender_id=defender_id,
                p=remaining_defender_power
            )
            outcome = "damaged"

        session.run(
            "MATCH (a:Unit {id: $attacker_id}) SET a.destructive_power_joule = $p",
            attacker_id=attacker_id,
            p=remaining_attacker_power
        )

        SimulationAction.objects.create(
            action_type="ENGAGE",
            unit_id=attacker_id,
            from_hex="N/A",
            to_hex="N/A",
            cost_joules=atk_power - remaining_attacker_power,
            remaining_power=remaining_attacker_power,
            timestamp=now()
        )

        return {
            "attacker_power_left": remaining_attacker_power,
            "defender_status": outcome,
            "damage_dealt": net_damage
        }

🔁 Future Extensions
Line-of-sight (LOS), elevation, and terrain modifiers

Combat morale & panic routes

Resupply and logistics

Retreat paths and fallback logic

Aggregated theater view with tactical layers

🗺️ Geo Infrastructure
All hexagons are H3 indexes

Road/rail connections stored in Neo4j as CONNECTED_TO

Units only move where connections exist

Hexagons can store terrain type, supply level, and visibility

graph TD
    A[Intelligence Scraper] --> B[LLM Extractor]
    B --> C[Parsed JSON Event]
    C --> D[Django Backend API]
    D --> E[Neo4j Simulation Graph]
    D --> F[PostgreSQL Logs]
    E --> G[Simulation Engine]
    G --> F
