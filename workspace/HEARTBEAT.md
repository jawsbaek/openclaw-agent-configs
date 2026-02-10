# HEARTBEAT.md - Rotating Heartbeat System

## 🎯 Philosophy

Batch background checks into a single heartbeat that rotates through tasks based on what's most overdue. Run on the **cheapest model possible** (Gemini Flash).

---

## 📋 Check Schedule

Each check has:
- **Cadence:** How often it should run
- **Time window:** When it's allowed to run (optional)
- **Last run:** Tracked in `memory/heartbeat-state.json`

### Current Checks

| Check | Cadence | Time Window | Purpose |
|-------|---------|-------------|---------|
| **Email** | 30 min | 09:00-21:00 | Urgent inbox scan |
| **Calendar** | 2 hours | 08:00-22:00 | Upcoming events |
| **Workspace** | 24 hours | Anytime | Git status, file cleanup |
| **Proactive** | 24 hours | 03:00-04:00 | Background maintenance |

---

## 🔄 How It Works

**On each heartbeat tick:**

1. **Load state:** Read `memory/heartbeat-state.json`
2. **Calculate overdue:** Find which check is most overdue (considering time windows)
3. **Run check:** Execute only the most overdue check
4. **Update timestamp:** Save to state file
5. **Report or silent:**
   - If actionable → Report to user
   - If nothing → Return `HEARTBEAT_OK`

---

## 📊 State File Format

```json
{
  "lastChecks": {
    "email": 1770541200,
    "calendar": 1770538800,
    "workspace": 1770490000,
    "proactive": 1770450000
  }
}
```

**Unix timestamps** (seconds since epoch).

---

## 🎯 Check Definitions

### 1. Email Check (30 min, 09:00-21:00)
```
- Check unread important emails
- Flag urgent messages
- Spawn email agent if action needed
```

### 2. Calendar Check (2 hours, 08:00-22:00)
```
- Events in next 24-48 hours
- Conflicts or scheduling issues
- Upcoming deadlines
```

### 3. Workspace Check (24 hours)
```
- Git status (uncommitted changes)
- memory/ folder (>7 days cleanup)
- reports/ archiving (>30 days)
```

### 4. Proactive Check (24 hours, 03:00-04:00)
```
- Review yesterday's memory
- Update MEMORY.md if needed
- System health (disk space, logs)
```

---

## ⚙️ Implementation Logic

**Pseudo-code:**
```python
def run_heartbeat():
    state = load_state()
    now = current_time()
    current_hour = now.hour
    
    eligible_checks = []
    
    for check in checks:
        if check.time_window:
            start, end = check.time_window
            if not (start <= current_hour < end):
                continue  # Outside time window
        
        last_run = state.get(check.name, 0)
        overdue_by = (now - last_run) - check.cadence
        
        if overdue_by > 0:
            eligible_checks.append((check, overdue_by))
    
    if not eligible_checks:
        return "HEARTBEAT_OK"
    
    # Run most overdue check
    most_overdue = max(eligible_checks, key=lambda x: x[1])
    check = most_overdue[0]
    
    result = execute_check(check)
    state[check.name] = now
    save_state(state)
    
    return result if result else "HEARTBEAT_OK"
```

---

## 🚫 What NOT to Do

❌ **Don't do work inline**
- Heartbeat = coordinator
- Spawn agents for actual work

❌ **Don't use expensive models**
- Heartbeat runs often
- Use Gemini Flash or GPT-5 Nano

❌ **Don't fire all checks at once**
- Rotate through checks
- One per heartbeat tick

---

## 📝 Execution Example

**Heartbeat tick at 10:30:**
```
1. Load state:
   - email: 10:00 (30 min ago)
   - calendar: 08:00 (2.5 hours ago) ← MOST OVERDUE
   - workspace: yesterday
   - proactive: yesterday (but outside 03:00-04:00 window)

2. Run calendar check:
   - Query events for next 48 hours
   - Found: Meeting at 14:00 today
   
3. Report:
   "Upcoming: Team sync at 14:00 (3.5 hours)"
   
4. Update state:
   calendar: 10:30
```

---

## 🔧 State File Maintenance

**Auto-create if missing:**
```json
{
  "lastChecks": {}
}
```

**Cleanup old entries:**
- Remove checks that no longer exist
- Reset to 0 if timestamp is invalid

---

## 💡 Tips

1. **Start conservative:** Long cadences, narrow time windows
2. **Monitor token usage:** Should be <$0.01/day for heartbeats
3. **Adjust based on needs:** More frequent email checks if critical
4. **Spawn agents:** Never do heavy work in heartbeat itself

---

## 📊 Expected Costs

| Check | Frequency | Tokens/day | Cost/month |
|-------|-----------|------------|------------|
| Email | 48/day | ~500 | $0.03 |
| Calendar | 12/day | ~300 | $0.01 |
| Workspace | 1/day | ~100 | $0.001 |
| Proactive | 1/day | ~200 | $0.002 |
| **Total** | **~62/day** | **~1100** | **~$0.05** |

*Based on Gemini Flash pricing (~$0.001/1K tokens)*

---

## 🎯 Success Metrics

- ✅ Heartbeat runs complete in <5 seconds
- ✅ Monthly cost <$0.10
- ✅ No missed critical emails/events
- ✅ Workspace stays clean (memory pruned, git clean)

---

**If this file changes, update the heartbeat prompt in your cron job or gateway config.**
