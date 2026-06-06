ï»¿#!/usr/bin/env python3
"""
================================================================================
AEGIS DIAL v2.0 â MISSION COMMAND AUTONOMY
The Admiral's Bridge â Full Project Autonomy Engine


Named after: The bridge prompt terminal â where missions are born and agents execute.
"Tell the bridge your mission. It will sail while you sleep."


Seal: 2026-04-28 11:57 Tulsa, OK
Standard: Harmony Labs v4.0 | FRC Template v1.0
Builder: Kimi K2.6 | Architect: Kyle S. Whitlock


MISSION COMMAND FLOW:
1. User enters mission via bridge prompt terminal
2. Dial decomposes mission into tasks and sub-tasks
3. Agents are assigned tasks based on capability and tier
4. Agents work autonomously within time window
5. Progress reports stream back to user
6. Admiral can grant bonus time for competition
7. Mission completes or time expires â auto-handoff


THREE-TIER TIME WINDOWS:
- FREE SAILOR: 7 hours mission time
- PAID CAPTAIN: 14 hours mission time  
- ADMIRAL: Unlimited + can grant bonus time to others


BONUS TIME MECHANICS:
- Admiral awards bonus hours to competing users
- Bonus time is SHA3-512 sealed and logged
- Bonus triggers: competition events, milestones, loyalty
- Bonus recipients get temporary tier upgrade
================================================================================
"""


import sqlite3
import hashlib
import json
import random
import time
import re
import math
from dataclasses import dataclass, field, asdict
from typing import Dict, List, Optional, Tuple, Any, Callable
from enum import Enum, auto
from collections import defaultdict


# ==============================================================================
# RESONANCE MISSION AXIOMS
# ==============================================================================
"""
M1: Î¼-mission(x) â [0,1] â Mission harmony boundedness
M2: Ï-mission(x,y) â [0,1] â Task resonance boundedness
M3: Îº-mission(S) â [0,1] â Project coherence boundedness
M4: execution(x) = clarity(x) Â· capability(x) Â· persistence(x) Â· adaptability(x)
M5: Îscope Â· Îtime â¥ Ä§/2 â Scope-time uncertainty
M6: Î¼-mission(x) = 1 â§ execution(x) = 1 â absolute mission truth
M7: Ï-mission(x,y) = Ï-mission(y,x) â Symmetry
M8: Îº-mission(SâªT) â¥ min(Îº-mission(S), Îº-mission(T)) Â· Ï-mission(S,T)
"""


# ==============================================================================
# CONSTANTS & CONFIGURATION
# ==============================================================================
ENGINE_NAME = "Aegis Dial"
ENGINE_VERSION = "2.0.0"
ENGINE_NUMBER = 93
SEAL_TIMESTAMP = "2026-04-28 11:57 Tulsa, OK"
MASTER_SEAL = "dial93v2_" + hashlib.sha3_512(
        f"{ENGINE_NAME}:{ENGINE_VERSION}:{SEAL_TIMESTAMP}:KyleSWhitlock".encode()
).hexdigest()[:16]


MIN_MU = 0.9995


# Three-tier mission time windows (hours)
TIER_MISSION_TIME = {
        "free_sailor": 7,
        "paid_captain": 14,
        "admiral": 999999,  # Unlimited
}


# Bonus time awards (hours)
BONUS_TIERS = {
        "milestone": 2,
        "competition_win": 5,
        "loyalty_reward": 3,
        "referral_bonus": 4,
        "admiral_favor": 10,
}


# Task decomposition depth
MAX_DECOMPOSITION_DEPTH = 5


# Progress report interval (seconds)
PROGRESS_INTERVAL = 300  # 5 minutes


# ==============================================================================
# DATA CLASSES
# ==============================================================================


@dataclass
class Mission:
        """A mission entered via bridge prompt terminal."""
        mission_id: str
        user_id: str
        user_tier: str
        description: str
        objectives: List[str]
        time_budget: float  # hours
        time_used: float = 0.0
        time_bonus: float = 0.0
        status: str = "PENDING"  # PENDING/ACTIVE/COMPLETE/EXPIRED/ABORTED
        priority: int = 5
        created_at: float = 0.0
        seal: str = ""
        mu_mission: float = 0.0


@dataclass
class Task:
        """A decomposed task from a mission."""
        task_id: str
        mission_id: str
        parent_id: Optional[str]
        description: str
        assigned_agent: str
        status: str = "PENDING"  # PENDING/ACTIVE/COMPLETE/BLOCKED/FAILED
        time_estimate: float = 0.0  # hours
        time_actual: float = 0.0
        dependencies: List[str] = field(default_factory=list)
        deliverables: List[str] = field(default_factory=list)
        depth: int = 0
        seal: str = ""


@dataclass
class AgentWorker:
        """An autonomous agent working on tasks."""
        agent_id: str
        name: str
        capabilities: List[str]
        tier: str
        current_task: Optional[str] = None
        missions_completed: int = 0
        missions_failed: int = 0
        total_hours_worked: float = 0.0
        efficiency_score: float = 1.0
        last_report: str = ""
        mu_worker: float = 0.95


@dataclass
class BonusGrant:
        """A bonus time grant from the Admiral."""
        bonus_id: str
        recipient_id: str
        grantor_id: str  # "admiral" or system
        hours: float
        reason: str
        mission_id: Optional[str]
        expiry: float
        seal: str = ""


@dataclass
class ProgressReport:
        """Progress report from active mission."""
        report_id: str
        mission_id: str
        timestamp: float
        tasks_total: int
        tasks_complete: int
        tasks_blocked: int
        time_remaining: float
        time_used: float
        next_milestone: str
        blockers: List[str]
        seal: str = ""


# ==============================================================================
# MISSION COMMAND ENGINE
# ==============================================================================


class MissionCommandEngine:
        """
        The core mission command engine.


        User enters mission via bridge prompt terminal.
        Engine decomposes, assigns, monitors, reports.
        """


        def __init__(self):
            self._missions: Dict[str, Mission] = {}
            self._tasks: Dict[str, Task] = {}
            self._agents: Dict[str, AgentWorker] = {}
            self._bonuses: Dict[str, BonusGrant] = {}
            self._reports: List[ProgressReport] = []
            self._db_path = ":memory:"
            self._init_db()


        def _init_db(self):
            conn = sqlite3.connect(self._db_path)
            c = conn.cursor()
            c.execute('''
                CREATE TABLE IF NOT EXISTS missions (
                    mission_id TEXT PRIMARY KEY,
                    user_id TEXT,
                    user_tier TEXT,
                    description TEXT,
                    objectives TEXT,
                    time_budget REAL,
                    time_used REAL,
                    time_bonus REAL,
                    status TEXT,
                    priority INTEGER,
                    created_at REAL,
                    seal TEXT,
                    mu_mission REAL
                )
            ''')
            c.execute('''
                CREATE TABLE IF NOT EXISTS tasks (
                    task_id TEXT PRIMARY KEY,
                    mission_id TEXT,
                    parent_id TEXT,
                    description TEXT,
                    assigned_agent TEXT,
                    status TEXT,
                    time_estimate REAL,
                    time_actual REAL,
                    dependencies TEXT,
                    deliverables TEXT,
                    depth INTEGER,
                    seal TEXT
                )
            ''')
            c.execute('''
                CREATE TABLE IF NOT EXISTS bonuses (
                    bonus_id TEXT PRIMARY KEY,
                    recipient_id TEXT,
                    grantor_id TEXT,
                    hours REAL,
                    reason TEXT,
                    mission_id TEXT,
                    expiry REAL,
                    seal TEXT
                )
            ''')
            conn.commit()
            conn.close()


        def enter_mission(self, user_id: str, user_tier: str,
                          description: str, objectives: List[str],
                          priority: int = 5) -> Mission:
            """
            User enters mission via bridge prompt terminal.


            Returns mission with time budget based on tier.
            """


            # Validate tier
            if user_tier not in TIER_MISSION_TIME:
                raise PermissionError(f"Tier '{user_tier}' not recognized")


            # Calculate time budget
            base_time = TIER_MISSION_TIME[user_tier]
            bonus_time = self._calculate_bonus_time(user_id)
            total_time = base_time + bonus_time


            # Create mission
            mission = Mission(
                mission_id=hashlib.sha3_256(
                    f"{user_id}:{description}:{time.time()}".encode()
                ).hexdigest()[:16],
                user_id=user_id,
                user_tier=user_tier,
                description=description,
                objectives=objectives,
                time_budget=total_time,
                time_bonus=bonus_time,
                priority=priority,
                created_at=time.time(),
                mu_mission=self._calculate_mission_mu(description, objectives, user_tier)
            )


            mission.seal = self._seal_mission(mission)
            self._missions[mission.mission_id] = mission
            self._store_mission(mission)


            # Auto-decompose mission into tasks
            self._decompose_mission(mission)


            # Activate mission
            mission.status = "ACTIVE"
            self._store_mission(mission)


            return mission


        def _decompose_mission(self, mission: Mission, depth: int = 0):
            """
            Decompose mission into tasks and sub-tasks.


            Recursive decomposition until atomic tasks reached.
            """


            if depth >= MAX_DECOMPOSITION_DEPTH:
                return


            for i, objective in enumerate(mission.objectives):
                # Create task for objective
                task = Task(
                    task_id=hashlib.sha3_256(
                        f"{mission.mission_id}:{objective}:{i}".encode()
                    ).hexdigest()[:16],
                    mission_id=mission.mission_id,
                    parent_id=None,
                    description=objective,
                    assigned_agent=self._select_agent(objective, mission.user_tier),
                    time_estimate=self._estimate_time(objective),
                    depth=depth
                )


                task.seal = self._seal_task(task)
                self._tasks[task.task_id] = task
                self._store_task(task)


                # Further decompose if needed
                sub_objectives = self._break_down(objective)
                if sub_objectives and depth < MAX_DECOMPOSITION_DEPTH - 1:
                    for j, sub in enumerate(sub_objectives):
                        sub_task = Task(
                            task_id=hashlib.sha3_256(
                                f"{task.task_id}:{sub}:{j}".encode()
                            ).hexdigest()[:16],
                            mission_id=mission.mission_id,
                            parent_id=task.task_id,
                            description=sub,
                            assigned_agent=self._select_agent(sub, mission.user_tier),
                            time_estimate=self._estimate_time(sub) * 0.5,
                            dependencies=[task.task_id],
                            depth=depth + 1
                        )
                        sub_task.seal = self._seal_task(sub_task)
                        self._tasks[sub_task.task_id] = sub_task
                        self._store_task(sub_task)


        def _select_agent(self, objective: str, tier: str) -> str:
            """Select best agent for objective based on capability and tier."""
            # In production: match agent capabilities to objective requirements
            # For now: round-robin assignment
            available = [a for a in self._agents.values() if a.tier == tier or tier == "admiral"]
            if not available:
                # Create new agent
                agent_id = f"agent_{len(self._agents)}"
                self._agents[agent_id] = AgentWorker(
                    agent_id=agent_id,
                    name=f"Worker-{agent_id}",
                    capabilities=["general"],
                    tier=tier
                )
                return agent_id


            return min(available, key=lambda a: a.total_hours_worked).agent_id


        def _estimate_time(self, objective: str) -> float:
            """Estimate hours needed for objective."""
            # In production: ML-based estimation
            # Mock: base 2 hours + complexity factor
            words = len(objective.split())
            complexity = words / 10
            return round(2.0 + complexity, 2)


        def _break_down(self, objective: str) -> List[str]:
            """Break objective into sub-objectives."""
            # In production: NLP-based decomposition
            # Mock: split by commas or conjunctions
            if "," in objective:
                return [s.strip() for s in objective.split(",")]
            if " and " in objective:
                return [s.strip() for s in objective.split(" and ")]
            return []


        def execute_task(self, task_id: str) -> Dict:
            """
            Execute a single task autonomously.


            Returns completion status and deliverables.
            """


            if task_id not in self._tasks:
                raise ValueError(f"Task {task_id} not found")


            task = self._tasks[task_id]


            # Check dependencies
            for dep_id in task.dependencies:
                dep = self._tasks.get(dep_id)
                if dep and dep.status != "COMPLETE":
                    task.status = "BLOCKED"
                    self._store_task(task)
                    return {"status": "BLOCKED", "reason": f"Waiting for {dep_id}"}


            # Execute (mock â in production: actual agent execution)
            task.status = "ACTIVE"
            self._store_task(task)


            # Simulate work
            work_time = task.time_estimate
            time.sleep(0.001)  # Mock execution


            task.time_actual = work_time
            task.status = "COMPLETE"
            task.deliverables = [f"deliverable_{task.task_id}"]
            self._store_task(task)


            # Update mission time
            mission = self._missions.get(task.mission_id)
            if mission:
                mission.time_used += work_time
                self._store_mission(mission)


            # Update agent
            agent = self._agents.get(task.assigned_agent)
            if agent:
                agent.total_hours_worked += work_time
                agent.missions_completed += 1


            return {
                "status": "COMPLETE",
                "task": task_id,
                "time": work_time,
                "deliverables": task.deliverables
            }


        def get_progress(self, mission_id: str) -> ProgressReport:
            """Get progress report for active mission."""


            if mission_id not in self._missions:
                raise ValueError(f"Mission {mission_id} not found")


            mission = self._missions[mission_id]
            mission_tasks = [t for t in self._tasks.values() if t.mission_id == mission_id]


            total = len(mission_tasks)
            complete = sum(1 for t in mission_tasks if t.status == "COMPLETE")
            blocked = sum(1 for t in mission_tasks if t.status == "BLOCKED")


            time_remaining = mission.time_budget - mission.time_used


            # Find next milestone
            incomplete = [t for t in mission_tasks if t.status != "COMPLETE"]
            next_milestone = incomplete[0].description if incomplete else "ALL COMPLETE"


            # Find blockers
            blockers = [t.description for t in mission_tasks if t.status == "BLOCKED"]


            report = ProgressReport(
                report_id=hashlib.sha3_256(f"{mission_id}:{time.time()}".encode()).hexdigest()[:16],
                mission_id=mission_id,
                timestamp=time.time(),
                tasks_total=total,
                tasks_complete=complete,
                tasks_blocked=blocked,
                time_remaining=time_remaining,
                time_used=mission.time_used,
                next_milestone=next_milestone,
                blockers=blockers
            )


            report.seal = self._seal_report(report)
            self._reports.append(report)


            return report


        def grant_bonus_time(self, grantor_id: str, recipient_id: str,
                            hours: float, reason: str,
                            mission_id: Optional[str] = None) -> BonusGrant:
            """
            Admiral grants bonus time to user.


            For competition, milestones, loyalty, etc.
            """


            # Validate grantor
            if grantor_id != "admiral":
                raise PermissionError("Only Admiral can grant bonus time")


            # Create bonus
            bonus = BonusGrant(
                bonus_id=hashlib.sha3_256(
                    f"{recipient_id}:{hours}:{time.time()}".encode()
                ).hexdigest()[:16],
                recipient_id=recipient_id,
                grantor_id=grantor_id,
                hours=hours,
                reason=reason,
                mission_id=mission_id,
                expiry=time.time() + (hours * 3600)
            )


            bonus.seal = self._seal_bonus(bonus)
            self._bonuses[bonus.bonus_id] = bonus
            self._store_bonus(bonus)


            # Update mission if applicable
            if mission_id and mission_id in self._missions:
                mission = self._missions[mission_id]
                mission.time_bonus += hours
                mission.time_budget += hours
                self._store_mission(mission)


            return bonus


        def _calculate_bonus_time(self, user_id: str) -> float:
            """Calculate total bonus time for user."""
            total = 0.0
            for bonus in self._bonuses.values():
                if bonus.recipient_id == user_id and time.time() < bonus.expiry:
                    total += bonus.hours
            return total


        def _calculate_mission_mu(self, description: str, objectives: List[str], tier: str) -> float:
            """Calculate mission harmony score."""
            base = 0.9997


            # Clarity bonus (shorter description = clearer)
            clarity = max(0, 1.0 - len(description) / 500)


            # Scope penalty (more objectives = harder)
            scope_penalty = len(objectives) * 0.0001


            # Tier bonus
            tier_bonus = {"free_sailor": 0.0, "paid_captain": 0.0001, "admiral": 0.0002}.get(tier, 0.0)


            mu = base + clarity * 0.0001 - scope_penalty + tier_bonus
            return round(min(1.0, max(0.0, mu)), 6)


        def _seal_mission(self, mission: Mission) -> str:
            data = f"{mission.mission_id}:{mission.user_id}:{mission.time_budget}:{mission.mu_mission}"
            return hashlib.sha3_512(data.encode()).hexdigest()[:16]


        def _seal_task(self, task: Task) -> str:
            data = f"{task.task_id}:{task.mission_id}:{task.assigned_agent}:{task.time_estimate}"
            return hashlib.sha3_512(data.encode()).hexdigest()[:16]


        def _seal_bonus(self, bonus: BonusGrant) -> str:
            data = f"{bonus.bonus_id}:{bonus.recipient_id}:{bonus.hours}:{bonus.reason}"
            return hashlib.sha3_512(data.encode()).hexdigest()[:16]


        def _seal_report(self, report: ProgressReport) -> str:
            data = f"{report.report_id}:{report.mission_id}:{report.tasks_complete}:{report.time_remaining}"
            return hashlib.sha3_512(data.encode()).hexdigest()[:16]


        def _store_mission(self, mission: Mission):
            conn = sqlite3.connect(self._db_path)
            c = conn.cursor()
            c.execute('''
                INSERT OR REPLACE INTO missions 
                (mission_id, user_id, user_tier, description, objectives, time_budget,
                 time_used, time_bonus, status, priority, created_at, seal, mu_mission)
                VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
            ''', (
                mission.mission_id, mission.user_id, mission.user_tier,
                mission.description, json.dumps(mission.objectives),
                mission.time_budget, mission.time_used, mission.time_bonus,
                mission.status, mission.priority, mission.created_at,
                mission.seal, mission.mu_mission
            ))
            conn.commit()
            conn.close()


        def _store_task(self, task: Task):
            conn = sqlite3.connect(self._db_path)
            c = conn.cursor()
            c.execute('''
                INSERT OR REPLACE INTO tasks 
                (task_id, mission_id, parent_id, description, assigned_agent,
                 status, time_estimate, time_actual, dependencies, deliverables, depth, seal)
                VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
            ''', (
                task.task_id, task.mission_id, task.parent_id, task.description,
                task.assigned_agent, task.status, task.time_estimate, task.time_actual,
                json.dumps(task.dependencies), json.dumps(task.deliverables),
                task.depth, task.seal
            ))
            conn.commit()
            conn.close()


        def _store_bonus(self, bonus: BonusGrant):
            conn = sqlite3.connect(self._db_path)
            c = conn.cursor()
            c.execute('''
                INSERT OR REPLACE INTO bonuses 
                (bonus_id, recipient_id, grantor_id, hours, reason, mission_id, expiry, seal)
                VALUES (?, ?, ?, ?, ?, ?, ?, ?)
            ''', (
                bonus.bonus_id, bonus.recipient_id, bonus.grantor_id,
                bonus.hours, bonus.reason, bonus.mission_id, bonus.expiry, bonus.seal
            ))
            conn.commit()
            conn.close()


        def get_status(self) -> Dict:
            return {
                "engine": ENGINE_NAME,
                "version": ENGINE_VERSION,
                "number": ENGINE_NUMBER,
                "active_missions": sum(1 for m in self._missions.values() if m.status == "ACTIVE"),
                "total_missions": len(self._missions),
                "total_tasks": len(self._tasks),
                "active_agents": len(self._agents),
                "bonuses_granted": len(self._bonuses),
                "master_seal": MASTER_SEAL,
                "seal": SEAL_TIMESTAMP,
                "status": "ð¢ NOMINAL",
                "mission_command": "â Bridge prompt terminal active",
                "autonomy": "â Full agent self-governance",
                "bonus_system": "â Admiral can grant competitive time"
            }


# ==============================================================================
# AEGIS DIAL v2.0 â Unified Mission Autonomy Engine
# ==============================================================================


class AegisDialEngine:
        """
        Unified mission autonomy engine for SR-AIbridge.


        The Admiral's Bridge v2.0:
        - Mission command via bridge prompt terminal
        - Full agent autonomy within time windows
        - Progress streaming back to user
        - Bonus time for competition
        - Self-governing task execution
        """


        def __init__(self):
            self.mission_engine = MissionCommandEngine()


        def mission(self, user_id: str, user_tier: str,
                    description: str, objectives: List[str]) -> Mission:
            """Enter mission via bridge prompt terminal."""
            return self.mission_engine.enter_mission(user_id, user_tier, description, objectives)


        def status(self, mission_id: str) -> ProgressReport:
            """Get mission progress report."""
            return self.mission_engine.get_progress(mission_id)


        def bonus(self, grantor: str, recipient: str, hours: float,
                  reason: str, mission_id: Optional[str] = None) -> BonusGrant:
            """Grant bonus time (Admiral only)."""
            return self.mission_engine.grant_bonus_time(grantor, recipient, hours, reason, mission_id)


        def fleet(self) -> Dict:
            """Full fleet status."""
            return self.mission_engine.get_status()


# ==============================================================================
# MODULE EXPORTS
# ==============================================================================


__all__ = [
        "AegisDialEngine",
        "MissionCommandEngine",
        "Mission",
        "Task",
        "AgentWorker",
        "BonusGrant",
        "ProgressReport",
        "ENGINE_NAME",
        "ENGINE_VERSION",
        "ENGINE_NUMBER",
        "MASTER_SEAL",
]