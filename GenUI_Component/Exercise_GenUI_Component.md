# Workshop: Building AI-Generated Financial Goal Trackers

## 🎯 What You'll Build

In this exercise, you'll extend an existing AI-powered banking application to support a new customer UI, **Financial Goal Trackers** - a new type of AI-generated widget that automatically tracks progress toward savings goals, debt payoff, and spending budgets.

By the end, users will be able to say things like:
- *"I want to save $5,000 for a vacation by December"*
- *"Help me track paying off my $3,000 credit card"*
- *"Set a $400 monthly budget for restaurants"*

And the AI will create dynamic, auto-updating goal trackers in their dashboard!

---

## 📋 Prerequisites

- The base banking application repository (with charts and simulators already working)
- Basic knowledge of:
  - Python/Flask (backend)
  - TypeScript/React (frontend)
  - SQL queries
  - How LangChain agents work

---

## 🏗️ Architecture Overview

Before we start coding, let's understand how the existing system works:

```
┌─────────────────────────────────────────────────────────────────────┐
│                         FRONTEND (React)                            │
├─────────────────────────────────────────────────────────────────────┤
│  ChatBot.tsx          │  AIModule.tsx        │  *Renderer.tsx       │
│  - User input         │  - Widget grid       │  - Visualization     │
│  - Send to backend    │  - Filter tabs       │  - Display data      │
│  - Show responses     │  - Refresh buttons   │                      │
└────────────┬──────────┴──────────┬───────────┴──────────────────────┘
             │                     │
             ▼                     ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         BACKEND (Flask)                             │
├─────────────────────────────────────────────────────────────────────┤
│  banking_app.py                                                     │
│  ├── AI Agent (LangChain)                                          │
│  │   └── Tools: get_accounts, transfer, create_widget, etc.        │
│  ├── API Endpoints: /api/ai-widgets, /api/ai-widgets/<id>/refresh  │
│  └── Database Models: User, Account, Transaction, AIWidget         │
├─────────────────────────────────────────────────────────────────────┤
│  widget_queries.py                                                  │
│  └── Query functions that fetch fresh data for dynamic widgets     │
├─────────────────────────────────────────────────────────────────────┤
│  ai_widget_model.py                                                 │
│  └── Database operations: create, update, delete widgets           │
└─────────────────────────────────────────────────────────────────────┘
```

### Key Concepts:

1. **Tools**: Functions the AI agent can call to perform actions
2. **Widget Types**: Different kinds of visualizations (chart, simulation, goal)
3. **Data Modes**: Static (snapshot) vs Dynamic (auto-refresh from database)
4. **Query Config**: Instructions for fetching fresh data for dynamic widgets

---

## 📚 Workshop Steps

| Step | File(s) | What You'll Learn |
|------|---------|-------------------|
| 1 | `src/types/aiModule.ts` | TypeScript interfaces and type safety |
| 2 | `backend/widget_queries.py` | Database queries and data aggregation |
| 3 | `backend/banking_app.py` | LangChain tools and agent prompts |
| 4 | `src/components/GoalWidgetRenderer.tsx` | React components and conditional rendering |
| 5 | `src/components/AIModule.tsx` | Component composition and filtering |
| 6 | `src/components/ChatBot.tsx` | User experience and feedback |

---

# Step 1: Define the Data Types

## 📁 File: `src/types/aiModule.ts`

### 🎓 Concept: TypeScript Interfaces

TypeScript interfaces define the "shape" of your data. They provide:
- **Autocomplete** in your IDE
- **Compile-time error checking**
- **Documentation** for other developers

### What to Add:

Add these new interfaces after the existing `SimulationConfig` interface:

```typescript
// ============================================
// GOAL WIDGET TYPES
// ============================================

export interface GoalConfig {
  goal_type: 'savings' | 'debt_payoff' | 'spending_limit';
  target_amount: number;
  deadline?: string;                    // ISO date string: "2025-12-31"
  linked_account_name?: string;         // Which account to track
  category?: string;                    // For spending_limit goals
  original_debt?: number;               // For debt_payoff goals
  spending_limit?: number;              // For spending_limit goals
}

export interface GoalProgressData {
  // Common fields
  progress_percent: number;
  
  // Savings goal fields
  current_amount?: number;
  target_amount?: number;
  remaining?: number;
  account_name?: string;
  recent_contributions?: number;
  
  // Debt payoff goal fields
  current_debt?: number;
  original_debt?: number;
  amount_paid?: number;
  
  // Spending limit goal fields
  current_spending?: number;
  spending_limit?: number;
  category?: string;
  is_over_budget?: boolean;
}
```

### Also Update Existing Interfaces:

Find the `WidgetConfig` interface and add `goalConfig`:

```typescript
export interface WidgetConfig {
  chartType?: 'line' | 'bar' | 'pie' | 'area' | 'scatter';
  // ... existing fields ...
  goalConfig?: GoalConfig;  // 👈 ADD THIS LINE
  customProps?: {
    data?: any[] | GoalProgressData;  // 👈 UPDATE THIS LINE
    [key: string]: any;
  };
}
```

Find the `QueryConfig` interface and add goal query types:

```typescript
export interface QueryConfig {
  query_type: 
    | 'spending_by_category' 
    | 'monthly_trend' 
    // ... existing types ...
    | 'goal_savings_progress'      // 👈 ADD
    | 'goal_debt_payoff_progress'  // 👈 ADD
    | 'goal_spending_limit';       // 👈 ADD
  // ... rest stays the same
}
```

### 💡 Why This Matters

By defining types first, we:
1. Have a clear contract between frontend and backend
2. Get IDE support while coding the rest
3. Catch type mismatches before runtime

---

# Step 2: Create Database Query Functions

## 📁 File: `backend/widget_queries.py`

### 🎓 Concept: Data Access Layer

The `widget_queries.py` file is our **data access layer**. It:
- Separates database logic from business logic
- Makes queries reusable across the application
- Handles the complexity of aggregating financial data

### Understanding the Existing Pattern

Look at the existing `execute_widget_query` function. It's a **dispatcher** that routes to specific query functions:

```python
def execute_widget_query(query_config: dict, user_id: str, db_session) -> list:
    query_type = query_config.get('query_type')
    
    if query_type == 'spending_by_category':
        return query_spending_by_category(...)
    elif query_type == 'monthly_trend':
        return query_monthly_trend(...)
    # ... etc
```

### What to Add:

#### 2.1 Add Goal Query Routing

Find the `execute_widget_query` function and add these cases before the `else: return []`:

```python
    # ============================================
    # GOAL QUERY TYPES
    # ============================================
    elif query_type == 'goal_savings_progress':
        return query_goal_savings_progress(user_id, query_config, db_session)
    elif query_type == 'goal_debt_payoff_progress':
        return query_goal_debt_payoff_progress(user_id, query_config, db_session)
    elif query_type == 'goal_spending_limit':
        return query_goal_spending_limit(user_id, start_date, end_date, query_config, db_session)
```

#### 2.2 Add Savings Goal Query Function

Add this function at the end of the file:

```python
def query_goal_savings_progress(user_id: str, query_config: dict, db_session) -> dict:
    """
    Query current progress toward a savings goal.
    
    This function:
    1. Finds the linked account (or sums all savings accounts)
    2. Calculates progress percentage
    3. Fetches recent contributions for context
    
    Returns a dict (not a list) because goals have single-value metrics,
    not arrays of data points like charts.
    """
    from banking_app import Account, Transaction, db
    
    filters = query_config.get('filters', {})
    target_amount = filters.get('target_amount', 0)
    linked_account_name = filters.get('linked_account_name')
    
    # Step 1: Get current balance
    if linked_account_name:
        # Track a specific account
        account = db_session.query(Account).filter(
            Account.name == linked_account_name,
            Account.user_id == user_id
        ).first()
        current_amount = account.balance if account else 0
        account_name = account.name if account else linked_account_name
        linked_account_id = account.id if account else None
    else:
        # Sum all savings accounts
        accounts = db_session.query(Account).filter(
            Account.user_id == user_id,
            Account.account_type == 'savings'
        ).all()
        current_amount = sum(acc.balance for acc in accounts)
        account_name = "All Savings"
        linked_account_id = None
    
    # Step 2: Calculate progress
    progress_percent = min(100, (current_amount / target_amount * 100)) if target_amount > 0 else 0
    remaining = max(0, target_amount - current_amount)
    
    # Step 3: Get recent contributions (last 30 days)
    recent_contributions = 0
    if linked_account_id:
        from datetime import datetime, timedelta
        thirty_days_ago = datetime.utcnow() - timedelta(days=30)
        
        contributions = db_session.query(
            db.func.sum(Transaction.amount)
        ).filter(
            Transaction.to_account_id == linked_account_id,
            Transaction.type.in_(['deposit', 'transfer']),
            Transaction.created_at >= thirty_days_ago
        ).scalar()
        recent_contributions = float(contributions) if contributions else 0
    
    return {
        'current_amount': round(current_amount, 2),
        'target_amount': round(target_amount, 2),
        'progress_percent': round(progress_percent, 1),
        'remaining': round(remaining, 2),
        'account_name': account_name,
        'recent_contributions': round(recent_contributions, 2),
    }
```

#### 2.3 Add Debt Payoff Query Function

```python
def query_goal_debt_payoff_progress(user_id: str, query_config: dict, db_session) -> dict:
    """
    Query progress toward paying off debt.
    
    Key insight: We track how much has been PAID OFF, not how much remains.
    This gives users a sense of progress rather than focusing on what's left.
    """
    from banking_app import Account, db
    
    filters = query_config.get('filters', {})
    original_debt = filters.get('original_debt', 0)
    linked_account_name = filters.get('linked_account_name')
    
    # Get current debt balance
    if linked_account_name:
        account = db_session.query(Account).filter(
            Account.name == linked_account_name,
            Account.user_id == user_id
        ).first()
        # Credit accounts typically have negative balances (amount owed)
        current_debt = abs(account.balance) if account else 0
        account_name = account.name if account else linked_account_name
    else:
        # Sum all credit accounts
        accounts = db_session.query(Account).filter(
            Account.user_id == user_id,
            Account.account_type == 'credit'
        ).all()
        current_debt = sum(abs(acc.balance) for acc in accounts)
        account_name = "All Credit"
    
    # Calculate progress (how much paid off)
    amount_paid = max(0, original_debt - current_debt)
    progress_percent = min(100, (amount_paid / original_debt * 100)) if original_debt > 0 else 0
    
    return {
        'current_debt': round(current_debt, 2),
        'original_debt': round(original_debt, 2),
        'amount_paid': round(amount_paid, 2),
        'progress_percent': round(progress_percent, 1),
        'account_name': account_name,
    }
```

#### 2.4 Add Spending Limit Query Function

```python
def query_goal_spending_limit(user_id: str, start_date: datetime, end_date: datetime, 
                               query_config: dict, db_session) -> dict:
    """
    Query spending against a budget limit.
    
    This is different from savings/debt because:
    - It uses a TIME RANGE (this month, this week, etc.)
    - Progress going UP is BAD (you're spending more)
    - We need to warn users when they're close to or over budget
    """
    from banking_app import Account, Transaction, db
    
    filters = query_config.get('filters', {})
    spending_limit = filters.get('spending_limit', 0)
    category = filters.get('category')  # Optional: specific category like "Restaurants"
    
    # Get user's account IDs
    account_ids = get_user_account_ids(user_id, db_session, filters)
    if not account_ids:
        return {
            'current_spending': 0,
            'spending_limit': spending_limit,
            'remaining': spending_limit,
            'progress_percent': 0,
            'category': category or 'All Categories',
            'is_over_budget': False
        }
    
    # Query spending in the time period
    query = db_session.query(
        db.func.sum(Transaction.amount)
    ).filter(
        Transaction.type == 'payment',
        Transaction.from_account_id.in_(account_ids),
        Transaction.created_at.between(start_date, end_date)
    )
    
    # Filter by category if specified
    if category:
        query = query.filter(Transaction.category == category)
    
    current_spending = query.scalar() or 0
    
    # Calculate metrics
    remaining = max(0, spending_limit - current_spending)
    progress_percent = min(100, (current_spending / spending_limit * 100)) if spending_limit > 0 else 0
    
    return {
        'current_spending': round(float(current_spending), 2),
        'spending_limit': round(spending_limit, 2),
        'remaining': round(remaining, 2),
        'progress_percent': round(progress_percent, 1),
        'category': category or 'All Categories',
        'is_over_budget': current_spending > spending_limit
    }
```

### 💡 Key Patterns to Notice

1. **Defensive coding**: Always handle missing data with defaults
2. **Type conversion**: Use `round()` for clean numbers, `float()` for Decimal types
3. **Different return shapes**: Charts return `list`, goals return `dict`

---

# Step 3: Create the AI Agent Tool

## 📁 File: `backend/banking_app.py`

### 🎓 Concept: LangChain Tools

A **Tool** is a function that the AI agent can call. The agent:
1. Reads the function's docstring to understand what it does
2. Looks at the parameters to know what information it needs
3. Decides when to call it based on user intent

The docstring is **critical** - it's how the AI learns to use your tool!

### What to Add:

#### 3.1 Create the Goal Tool Function

Find the `create_simulation_widget_for_current_user` function and add this new function after it:

```python
# ============================================
# GOAL WIDGET TOOL
# ============================================

def create_goal_widget_for_current_user(
    title: str,
    goal_type: str = "savings",
    target_amount: float = 1000,
    deadline: str = None,
    description: str = "",
    linked_account_name: str = None,
    category: str = None,
    original_debt: float = None,
    spending_limit: float = None,
    time_range: str = "this_month"
) -> str:
    """
    Creates a financial goal tracker widget in the user's AI Module dashboard.
    Use this tool when the user wants to set savings goals, debt payoff targets, or spending limits.
    
    These widgets automatically update based on the user's account balances and transactions.
    
    Args:
        title: Title for the goal (e.g., "Vacation Fund", "Pay Off Credit Card")
        goal_type: Type of goal. Must be one of:
            - 'savings': Track progress toward a savings target (e.g., vacation, emergency fund)
            - 'debt_payoff': Track progress paying off debt (credit cards, loans)
            - 'spending_limit': Track spending against a budget limit for a category
        target_amount: The target amount to reach (for savings goals)
        deadline: Optional deadline date in YYYY-MM-DD format (e.g., "2025-12-31")
        description: Description of the goal
        linked_account_name: Name of the account to track (e.g., "Vacation Savings")
        category: For spending_limit goals, the category to track (e.g., "Restaurants")
        original_debt: For debt_payoff goals, the original debt amount
        spending_limit: For spending_limit goals, the monthly budget limit
        time_range: For spending_limit goals: 'this_month', 'this_week', 'last_30_days'
    
    Returns:
        JSON string with status and widget_id if successful
    
    Examples:
        - "Save $5000 for vacation by December" -> goal_type='savings', target_amount=5000
        - "Pay off my $3000 credit card" -> goal_type='debt_payoff', original_debt=3000
        - "Keep restaurant spending under $400/month" -> goal_type='spending_limit', spending_limit=400, category='Restaurants'
    """
    from ai_widget_model import create_widget
    
    try:
        # Validate goal type
        valid_types = ['savings', 'debt_payoff', 'spending_limit']
        if goal_type not in valid_types:
            return json.dumps({
                "status": "error",
                "message": f"Invalid goal_type. Must be one of: {', '.join(valid_types)}"
            })
        
        # Build query config based on goal type
        query_config = {
            "filters": {}
        }
        
        if goal_type == 'savings':
            query_config["query_type"] = "goal_savings_progress"
            query_config["filters"]["target_amount"] = target_amount
            if linked_account_name:
                query_config["filters"]["linked_account_name"] = linked_account_name
                
        elif goal_type == 'debt_payoff':
            query_config["query_type"] = "goal_debt_payoff_progress"
            query_config["filters"]["original_debt"] = original_debt or target_amount
            if linked_account_name:
                query_config["filters"]["linked_account_name"] = linked_account_name
                
        elif goal_type == 'spending_limit':
            query_config["query_type"] = "goal_spending_limit"
            query_config["time_range"] = time_range
            query_config["filters"]["spending_limit"] = spending_limit or target_amount
            if category:
                query_config["filters"]["category"] = category
        
        # Build goal config (stored in widget for display purposes)
        goal_config = {
            "goal_type": goal_type,
            "target_amount": target_amount,
            "deadline": deadline,
            "linked_account_name": linked_account_name,
            "category": category,
            "original_debt": original_debt,
            "spending_limit": spending_limit,
        }
        
        # Build widget config
        config = {
            "goalConfig": goal_config,
            "customProps": {}
        }
        
        # Fetch initial data immediately
        try:
            initial_data = execute_widget_query(query_config, user_id, db.session)
            config["customProps"]["data"] = initial_data
        except Exception as e:
            print(f"[goal_widget] Error fetching initial data: {e}")
            config["customProps"]["data"] = {}
        
        # Create the widget in database
        widget = create_widget(
            user_id=user_id,
            title=title,
            description=description,
            widget_type="goal",          # 👈 New widget type!
            config=config,
            code=None,
            data_mode="dynamic",         # 👈 Goals are always dynamic
            query_config=query_config
        )
        
        # Build a helpful response message
        type_messages = {
            'savings': f"I've created a savings goal tracker for '{title}'. It will automatically update as your account balance changes.",
            'debt_payoff': f"I've created a debt payoff tracker for '{title}'. It will update as you pay down your balance.",
            'spending_limit': f"I've created a spending tracker for '{title}'. It will monitor your {category or 'overall'} spending against your ${spending_limit or target_amount} limit.",
        }
        
        return json.dumps({
            "status": "success",
            "message": type_messages.get(goal_type),
            "widget_id": widget['id'],
            "widget_type": "goal",
            "goal_type": goal_type
        })
        
    except Exception as e:
        traceback.print_exc()
        return json.dumps({
            "status": "error",
            "message": f"Failed to create goal widget: {str(e)}"
        })
```

#### 3.2 Register the Tool with the Agent

Find the `tools` list in the `chatbot()` function and add the new tool:

```python
tools = [
    get_user_accounts_for_current_user,
    get_transactions_summary_for_current_user,
    search_support_documents, 
    create_new_account_for_current_user,
    transfer_money_for_current_user,
    query_database,
    create_ai_widget_for_current_user,
    create_simulation_widget_for_current_user,
    update_ai_widget_for_current_user,
    create_goal_widget_for_current_user,  # 👈 ADD THIS
]
```

#### 3.3 Update the Agent's System Prompt

Find the `banking_agent = create_react_agent(...)` call and add goal instructions to the prompt. Add this section after the simulation instructions:

```python
## FINANCIAL GOAL WIDGETS ##
When users want to track financial goals, use `create_goal_widget_for_current_user`:

1. **savings**: For saving toward a target amount
   - "Save $5000 for vacation" -> goal_type='savings', target_amount=5000
   - Can link to specific account: linked_account_name='Vacation Savings'

2. **debt_payoff**: For paying off debt
   - "Help me pay off my $3000 credit card" -> goal_type='debt_payoff', original_debt=3000

3. **spending_limit**: For budget tracking
   - "Keep restaurant spending under $400" -> goal_type='spending_limit', spending_limit=400, category='Restaurants'

**Key differences from simulators:**
- Goals are ALWAYS dynamic (auto-update from real account data)
- Simulators are for "what-if" planning with adjustable inputs
- Goals track ACTUAL progress toward real financial targets
```

#### 3.4 Update the Response to Include Goal Type

Find where the chatbot response is built (near the end of the `chatbot()` function) and update it:

```python
# Check if a goal was created
goal_created = False
goal_type = None
for msg in final_messages:
    if hasattr(msg, 'content') and isinstance(msg.content, str):
        if '"widget_type": "goal"' in msg.content:
            goal_created = True
            # Extract goal_type from response
            if '"goal_type": "savings"' in msg.content:
                goal_type = 'savings'
            elif '"goal_type": "debt_payoff"' in msg.content:
                goal_type = 'debt_payoff'
            elif '"goal_type": "spending_limit"' in msg.content:
                goal_type = 'spending_limit'
            break
```
Now, add the following lines in the same function:
```python
return jsonify({
    "response": final_messages[-1].content,
    "session_id": session_id,
    "tools_used": [],
    "widget_created": widget_created or goal_created,  # 👈 UPDATE
    "widget_updated": widget_updated,
    "widget_mode": widget_mode,
    "widget_type": 'goal' if goal_created else widget_type,  # 👈 UPDATE
    "simulation_type": simulation_type,
    "goal_type": goal_type,  # 👈 ADD
})
```

### 💡 Key Patterns to Notice

1. **Docstrings are documentation for the AI**: Write them clearly with examples
2. **Always return JSON strings from tools**: The agent parses these
3. **Validate inputs early**: Fail fast with helpful error messages
4. **Query data immediately**: Don't make users refresh manually

---

# Step 4: Build the Goal Renderer Component

## 📁 File: `src/components/GoalWidgetRenderer.tsx` (NEW FILE)

### 🎓 Concept: React Component Composition

This component:
- Receives widget data as props
- Renders different UIs based on `goal_type`
- Uses conditional styling for visual feedback (green = good, red = warning)

### Create the New File:

```typescript
import React from 'react';
import { 
  Target, TrendingUp, TrendingDown, CreditCard, DollarSign, 
  Calendar, AlertTriangle, CheckCircle2, Wallet 
} from 'lucide-react';
import type { AIWidget, GoalConfig, GoalProgressData } from '../types/aiModule';

interface GoalWidgetRendererProps {
  widget: AIWidget;
}

const GoalWidgetRenderer: React.FC<GoalWidgetRendererProps> = ({ widget }) => {
  // Extract configuration and data from widget
  const goalConfig = widget.config.goalConfig as GoalConfig;
  const data = widget.config.customProps?.data as GoalProgressData;
  
  // Handle missing data gracefully
  if (!goalConfig || !data) {
    return (
      <div className="flex flex-col items-center justify-center h-full text-gray-400">
        <Target className="h-8 w-8 mb-2" />
        <p className="text-sm">No goal data available</p>
      </div>
    );
  }

  const { goal_type, deadline } = goalConfig;
  const progress = data.progress_percent || 0;
  
  // Helper: Calculate days until deadline
  const getDaysRemaining = () => {
    if (!deadline) return null;
    const deadlineDate = new Date(deadline);
    const today = new Date();
    const diffTime = deadlineDate.getTime() - today.getTime();
    return Math.ceil(diffTime / (1000 * 60 * 60 * 24));
  };
  
  const daysRemaining = getDaysRemaining();
  
  // Helper: Get progress bar color based on context
  const getProgressColor = () => {
    if (goal_type === 'spending_limit') {
      // For spending: more progress = BAD
      if (progress >= 100) return 'bg-red-500';
      if (progress >= 80) return 'bg-amber-500';
      return 'bg-green-500';
    }
    // For savings/debt: more progress = GOOD
    if (progress >= 100) return 'bg-green-500';
    if (progress >= 50) return 'bg-blue-500';
    return 'bg-indigo-500';
  };

  // ============================================
  // SAVINGS GOAL RENDERER
  // ============================================
  const renderSavingsGoal = () => (
    <div className="h-full flex flex-col">
      {/* Stats Cards */}
      <div className="grid grid-cols-2 gap-4 mb-6">
        <div className="bg-gradient-to-br from-green-50 to-emerald-50 rounded-xl p-4 border border-green-100">
          <div className="flex items-center gap-2 mb-1">
            <Wallet className="h-4 w-4 text-green-600" />
            <span className="text-xs font-medium text-green-600">Current Savings</span>
          </div>
          <p className="text-2xl font-bold text-green-700">
            ${(data.current_amount || 0).toLocaleString()}
          </p>
          {data.account_name && (
            <p className="text-xs text-green-600 mt-1">{data.account_name}</p>
          )}
        </div>
        
        <div className="bg-gradient-to-br from-blue-50 to-indigo-50 rounded-xl p-4 border border-blue-100">
          <div className="flex items-center gap-2 mb-1">
            <Target className="h-4 w-4 text-blue-600" />
            <span className="text-xs font-medium text-blue-600">Target</span>
          </div>
          <p className="text-2xl font-bold text-blue-700">
            ${(data.target_amount || 0).toLocaleString()}
          </p>
          <p className="text-xs text-blue-600 mt-1">
            ${(data.remaining || 0).toLocaleString()} to go
          </p>
        </div>
      </div>
      
      {/* Progress Bar */}
      <div className="mb-6">
        <div className="flex justify-between items-center mb-2">
          <span className="text-sm font-medium text-gray-700">Progress</span>
          <span className={`text-sm font-bold ${progress >= 100 ? 'text-green-600' : 'text-blue-600'}`}>
            {progress.toFixed(1)}%
          </span>
        </div>
        <div className="h-4 rounded-full bg-gray-100 overflow-hidden">
          <div 
            className={`h-full rounded-full transition-all duration-500 ${getProgressColor()}`}
            style={{ width: `${Math.min(100, progress)}%` }}
          />
        </div>
      </div>
      
      {/* Additional Context */}
      <div className="grid grid-cols-2 gap-4 mt-auto">
        {data.recent_contributions !== undefined && data.recent_contributions > 0 && (
          <div className="bg-gray-50 rounded-lg p-3">
            <div className="flex items-center gap-2">
              <TrendingUp className="h-4 w-4 text-green-500" />
              <span className="text-xs text-gray-500">Last 30 Days</span>
            </div>
            <p className="text-lg font-semibold text-gray-900 mt-1">
              +${data.recent_contributions.toLocaleString()}
            </p>
          </div>
        )}
        
        {daysRemaining !== null && (
          <div className="bg-gray-50 rounded-lg p-3">
            <div className="flex items-center gap-2">
              <Calendar className="h-4 w-4 text-indigo-500" />
              <span className="text-xs text-gray-500">Deadline</span>
            </div>
            <p className={`text-lg font-semibold mt-1 ${
              daysRemaining < 30 ? 'text-amber-600' : 'text-gray-900'
            }`}>
              {daysRemaining > 0 ? `${daysRemaining} days` : 'Past due'}
            </p>
          </div>
        )}
      </div>
      
      {/* Celebration when goal is reached! */}
      {progress >= 100 && (
        <div className="mt-4 bg-gradient-to-r from-green-100 to-emerald-100 rounded-xl p-4 border border-green-200 text-center">
          <CheckCircle2 className="h-8 w-8 text-green-600 mx-auto mb-2" />
          <p className="font-semibold text-green-800">🎉 Goal Achieved!</p>
          <p className="text-sm text-green-600">Congratulations on reaching your target!</p>
        </div>
      )}
    </div>
  );

  // ============================================
  // DEBT PAYOFF GOAL RENDERER
  // ============================================
  const renderDebtPayoffGoal = () => (
    <div className="h-full flex flex-col">
      <div className="grid grid-cols-2 gap-4 mb-6">
        <div className="bg-gradient-to-br from-red-50 to-orange-50 rounded-xl p-4 border border-red-100">
          <div className="flex items-center gap-2 mb-1">
            <CreditCard className="h-4 w-4 text-red-600" />
            <span className="text-xs font-medium text-red-600">Remaining Debt</span>
          </div>
          <p className="text-2xl font-bold text-red-700">
            ${(data.current_debt || 0).toLocaleString()}
          </p>
        </div>
        
        <div className="bg-gradient-to-br from-green-50 to-teal-50 rounded-xl p-4 border border-green-100">
          <div className="flex items-center gap-2 mb-1">
            <TrendingDown className="h-4 w-4 text-green-600" />
            <span className="text-xs font-medium text-green-600">Paid Off</span>
          </div>
          <p className="text-2xl font-bold text-green-700">
            ${(data.amount_paid || 0).toLocaleString()}
          </p>
          <p className="text-xs text-green-600 mt-1">
            of ${(data.original_debt || 0).toLocaleString()}
          </p>
        </div>
      </div>
      
      {/* Progress Bar */}
      <div className="mb-6">
        <div className="flex justify-between items-center mb-2">
          <span className="text-sm font-medium text-gray-700">Payoff Progress</span>
          <span className={`text-sm font-bold ${progress >= 100 ? 'text-green-600' : 'text-blue-600'}`}>
            {progress.toFixed(1)}%
          </span>
        </div>
        <div className="h-4 rounded-full bg-gray-100 overflow-hidden">
          <div 
            className={`h-full rounded-full transition-all duration-500 ${getProgressColor()}`}
            style={{ width: `${Math.min(100, progress)}%` }}
          />
        </div>
      </div>
      
      {/* Celebration */}
      {progress >= 100 && (
        <div className="mt-auto bg-gradient-to-r from-green-100 to-emerald-100 rounded-xl p-4 border border-green-200 text-center">
          <CheckCircle2 className="h-8 w-8 text-green-600 mx-auto mb-2" />
          <p className="font-semibold text-green-800">🎉 Debt Free!</p>
          <p className="text-sm text-green-600">You've paid off this debt completely!</p>
        </div>
      )}
    </div>
  );

  // ============================================
  // SPENDING LIMIT GOAL RENDERER
  // ============================================
  const renderSpendingLimitGoal = () => {
    const isOverBudget = data.is_over_budget || progress > 100;
    
    return (
      <div className="h-full flex flex-col">
        <div className="grid grid-cols-2 gap-4 mb-6">
          <div className={`rounded-xl p-4 border ${
            isOverBudget 
              ? 'bg-gradient-to-br from-red-50 to-orange-50 border-red-100' 
              : 'bg-gradient-to-br from-blue-50 to-indigo-50 border-blue-100'
          }`}>
            <div className="flex items-center gap-2 mb-1">
              <DollarSign className={`h-4 w-4 ${isOverBudget ? 'text-red-600' : 'text-blue-600'}`} />
              <span className={`text-xs font-medium ${isOverBudget ? 'text-red-600' : 'text-blue-600'}`}>
                Spent
              </span>
            </div>
            <p className={`text-2xl font-bold ${isOverBudget ? 'text-red-700' : 'text-blue-700'}`}>
              ${(data.current_spending || 0).toLocaleString()}
            </p>
            {data.category && (
              <p className={`text-xs mt-1 ${isOverBudget ? 'text-red-600' : 'text-blue-600'}`}>
                on {data.category}
              </p>
            )}
          </div>
          
          <div className="bg-gradient-to-br from-gray-50 to-slate-50 rounded-xl p-4 border border-gray-200">
            <div className="flex items-center gap-2 mb-1">
              <Target className="h-4 w-4 text-gray-600" />
              <span className="text-xs font-medium text-gray-600">Budget</span>
            </div>
            <p className="text-2xl font-bold text-gray-700">
              ${(data.spending_limit || 0).toLocaleString()}
            </p>
            <p className={`text-xs mt-1 ${isOverBudget ? 'text-red-600' : 'text-green-600'}`}>
              {isOverBudget 
                ? `$${((data.current_spending || 0) - (data.spending_limit || 0)).toLocaleString()} over`
                : `$${(data.remaining || 0).toLocaleString()} left`
              }
            </p>
          </div>
        </div>
        
        {/* Progress Bar */}
        <div className="mb-6">
          <div className="flex justify-between items-center mb-2">
            <span className="text-sm font-medium text-gray-700">Budget Used</span>
            <span className={`text-sm font-bold ${
              isOverBudget ? 'text-red-600' : progress >= 80 ? 'text-amber-600' : 'text-green-600'
            }`}>
              {progress.toFixed(1)}%
            </span>
          </div>
          <div className="h-4 rounded-full bg-gray-100 overflow-hidden">
            <div 
              className={`h-full rounded-full transition-all duration-500 ${getProgressColor()}`}
              style={{ width: `${Math.min(100, progress)}%` }}
            />
          </div>
        </div>
        
        {/* Status Message */}
        {isOverBudget ? (
          <div className="mt-auto bg-gradient-to-r from-red-100 to-orange-100 rounded-xl p-4 border border-red-200">
            <div className="flex items-center gap-3">
              <AlertTriangle className="h-6 w-6 text-red-600" />
              <div>
                <p className="font-semibold text-red-800">Over Budget!</p>
                <p className="text-sm text-red-600">
                  You've exceeded your {data.category || 'spending'} limit
                </p>
              </div>
            </div>
          </div>
        ) : progress >= 80 ? (
          <div className="mt-auto bg-gradient-to-r from-amber-50 to-yellow-50 rounded-xl p-4 border border-amber-200">
            <div className="flex items-center gap-3">
              <AlertTriangle className="h-6 w-6 text-amber-600" />
              <div>
                <p className="font-semibold text-amber-800">Approaching Limit</p>
                <p className="text-sm text-amber-600">
                  Only ${(data.remaining || 0).toLocaleString()} left in your budget
                </p>
              </div>
            </div>
          </div>
        ) : (
          <div className="mt-auto bg-gradient-to-r from-green-50 to-emerald-50 rounded-xl p-4 border border-green-200">
            <div className="flex items-center gap-3">
              <CheckCircle2 className="h-6 w-6 text-green-600" />
              <div>
                <p className="font-semibold text-green-800">On Track</p>
                <p className="text-sm text-green-600">
                  You have ${(data.remaining || 0).toLocaleString()} remaining
                </p>
              </div>
            </div>
          </div>
        )}
      </div>
    );
  };

  // ============================================
  // MAIN RENDER - Route to correct sub-renderer
  // ============================================
  switch (goal_type) {
    case 'savings':
      return renderSavingsGoal();
    case 'debt_payoff':
      return renderDebtPayoffGoal();
    case 'spending_limit':
      return renderSpendingLimitGoal();
    default:
      return (
        <div className="flex flex-col items-center justify-center h-full text-gray-400">
          <Target className="h-8 w-8 mb-2" />
          <p className="text-sm">Unknown goal type: {goal_type}</p>
        </div>
      );
  }
};

export default GoalWidgetRenderer;
```

### 💡 Key Patterns to Notice

1. **Render functions for each variant**: Keeps code organized and readable
2. **Semantic colors**: Green = good, Red = warning, Blue = info
3. **Progress visualization**: Users see both numbers AND visual progress
4. **Celebration states**: Positive reinforcement when goals are achieved!

---

# Step 5: Integrate Goals into the AI Module

## 📁 File: `src/components/AIModule.tsx`

### 🎓 Concept: Component Integration

Now we need to:
1. Import the new renderer
2. Add a filter tab for goals
3. Apply goal-specific styling
4. Route to the correct renderer

### What to Change:

#### 5.1 Add Imports

At the top of the file, add:

```typescript
import GoalWidgetRenderer from './GoalWidgetRenderer';
import { Target } from 'lucide-react';  // Add to existing lucide imports
```

#### 5.2 Update Filter State

Change the filter state type:

```typescript
// FROM:
const [activeFilter, setActiveFilter] = useState<'all' | 'charts' | 'simulations'>('all');

// TO:
const [activeFilter, setActiveFilter] = useState<'all' | 'charts' | 'simulations' | 'goals'>('all');
```

#### 5.3 Update Filter Logic

```typescript
// Update filteredWidgets
const filteredWidgets = widgets.filter(widget => {
  if (activeFilter === 'all') return true;
  if (activeFilter === 'simulations') return widget.widget_type === 'simulation';
  if (activeFilter === 'goals') return widget.widget_type === 'goal';  // 👈 ADD
  if (activeFilter === 'charts') return widget.widget_type !== 'simulation' && widget.widget_type !== 'goal';  // 👈 UPDATE
  return true;
});

// Add goal count
const chartCount = widgets.filter(w => w.widget_type !== 'simulation' && w.widget_type !== 'goal').length;
const simulationCount = widgets.filter(w => w.widget_type === 'simulation').length;
const goalCount = widgets.filter(w => w.widget_type === 'goal').length;  // 👈 ADD
```

#### 5.4 Add Goals Filter Tab

In the filter tabs section, add a new button:

```typescript
<button
  onClick={() => setActiveFilter('goals')}
  className={`flex items-center gap-2 px-4 py-2 rounded-lg text-sm font-medium transition-all ${
    activeFilter === 'goals' 
      ? 'bg-green-100 text-green-700' 
      : 'text-gray-600 hover:bg-gray-100'
  }`}
>
  <Target className="h-4 w-4" />
  Goals ({goalCount})
</button>
```

#### 5.5 Update Widget Card Styling

In the widget rendering loop, add goal detection and styling:

```typescript
const isRefreshing = refreshingWidgets.has(widget.id);
// Add detection there
const isGoal = widget.widget_type === 'goal';

// Update border color
className={`bg-white rounded-xl shadow-sm border overflow-hidden transition-all hover:shadow-md ${
  expandedWidget === widget.id ? 'col-span-full' : ''
} ${isSimulation ? 'border-amber-200' : isGoal ? 'border-green-200' : 'border-gray-200'}`}  // 👈 Update here

// Update header gradient
<div className={`px-6 py-4 border-b flex items-center justify-between ${
  isSimulation 
    ? 'bg-gradient-to-r from-amber-50 to-orange-50 border-amber-100' 
    : isGoal // 👈 ADD
      ? 'bg-gradient-to-r from-green-50 to-emerald-50 border-green-100' // 👈 ADD
      : 'bg-gradient-to-r from-gray-50 to-white border-gray-100' // 👈 ADD
}`}>

// Update icon
{isSimulation ? (
  <Sliders className="h-4 w-4 text-amber-600" />
) : isGoal ? (
  <Target className="h-4 w-4 text-green-600" />  // 👈 ADD
) : (
  <SparklesIcon size={16} />
)}

// Add goal badge
{isSimulation ? (
  <span className="inline-flex items-center gap-1 px-2 py-0.5 rounded-full text-xs font-medium bg-amber-50 text-amber-700 border border-amber-200">
    <Sliders className="h-3 w-3" />
    Interactive
  </span>
//  New ADD
) : isGoal ? (
  <span className="inline-flex items-center gap-1 px-2 py-0.5 rounded-full text-xs font-medium bg-green-50 text-green-700 border border-green-200">
    <Target className="h-3 w-3" />
    Goal
  </span>
//  New END
) : isDynamic ? (
  <span className="inline-flex items-center gap-1 px-2 py-0.5 rounded-full text-xs font-medium bg-cyan-50 text-cyan-700 border border-cyan-200">
    <Zap className="h-3 w-3" />
    Live
</span>
```

#### 5.6 Update Widget Content Height and Renderer

```typescript
// Update height
<div className={`p-6 relative ${
  expandedWidget === widget.id 
    ? 'h-[600px]' 
    : isSimulation 
      ? 'h-[480px]' 
      : isGoal.  // 👈 ADD
        ? 'h-[320px]'  // 👈 ADD
        : 'h-80'
}`}>

// Update renderer routing
{isSimulation ? (
  <SimulationWidgetRenderer widget={widget} />
) : isGoal ? ( // 👈 ADD
  <GoalWidgetRenderer widget={widget} />  // 👈 ADD
) : (
  <AIWidgetRenderer 
    widget={widget} 
    data={widget.config.customProps?.data}
  />
)}
```

#### 5.7 Enable Refresh for Goals

Goals are dynamic, so they need the refresh button. Update the condition:

```typescript
{/* Refresh button for dynamic widgets AND goals */}
{(isDynamic || isGoal) && !isSimulation && ( // 👈 Update here
  <button
    onClick={() => handleRefreshWidget(widget.id)}
    // ... rest stays the same
  >
```

---

# Step 6: Update the Chat Interface

## 📁 File: `src/components/ChatBot.tsx`

### 🎓 Concept: User Experience Feedback

Users need to know when their goal was created successfully. We'll:
1. Add goal-specific quick suggestions
2. Show a goal creation confirmation
3. Handle goal editing context

### What to Change:

#### 6.1 Update Message Interface

```typescript
interface Message {
  // ... existing fields
  widgetType?: 'chart' | 'simulation' | 'goal';  // 👈 Add 'goal'
  goalType?: string;  // 👈 ADD
}
```

#### 6.2 Add Goal Icon Helper

After ***getSimulationIcon***, add:

```typescript
const getGoalIcon = (goalType: string | undefined) => {
  switch (goalType) {
    case 'savings': 
      return { icon: PiggyBank, color: 'text-green-600', bg: 'bg-green-50', border: 'border-green-200', label: 'Savings Goal' };
    case 'debt_payoff': 
      return { icon: CreditCard, color: 'text-red-600', bg: 'bg-red-50', border: 'border-red-200', label: 'Debt Payoff Goal' };
    case 'spending_limit': 
      return { icon: Wallet, color: 'text-amber-600', bg: 'bg-amber-50', border: 'border-amber-200', label: 'Spending Budget' };
    default: 
      return { icon: Target, color: 'text-green-600', bg: 'bg-green-50', border: 'border-green-200', label: 'Goal Tracker' };
  }
};
```

#### 6.3 Update Quick Suggestions

```typescript
const quickSuggestions = activeTab === 'ai-module' ? [
  { text: 'Create a spending by category chart', icon: PieChart, color: 'text-blue-600' },
  { text: 'Save $5000 for a vacation', icon: Target, color: 'text-green-600' },  // 👈 ADD
  { text: 'Build a loan repayment calculator', icon: Home, color: 'text-amber-600' },
  { text: 'Track my restaurant spending budget', icon: Wallet, color: 'text-red-600' },  // 👈 ADD
] : [
  // ... default suggestions
];
```

#### 6.4 Add Goal Creation Indicator

In the message rendering, add after the simulation indicator:

```typescript
<div className="flex-1 overflow-y-auto p-4 space-y-4 bg-gray-50">
  {messages.map((message) => (
    <div
      key={message.id}
      className={`flex ${message.role === 'user' ? 'justify-end' : 'justify-start'}`}
    >
      <div
        className={`max-w-[85%] rounded-2xl px-4 py-2.5 ${
          message.role === 'user'
            ? 'bg-blue-600 text-white rounded-br-md'
            : 'bg-white text-gray-800 shadow-sm border border-gray-100 rounded-bl-md'
        }`}
      >
        <div className="flex items-start gap-2">
          {message.role === 'assistant' && (
            <Bot className="h-4 w-4 mt-0.5 text-blue-600 flex-shrink-0" />
          )}
          <div className="flex-1">
            <p className="text-sm whitespace-pre-wrap">{message.content}</p>

          {/* Goal creation indicator (NEW) */}
          {message.widgetCreated && message.widgetType === 'goal' && (
            <div className={`mt-2 p-2 rounded-lg text-xs ${getGoalIcon(message.goalType).bg} border ${getGoalIcon(message.goalType).border}`}>
              <div className="flex items-center gap-1.5">
                <Target className={`h-3.5 w-3.5 ${getGoalIcon(message.goalType).color}`} />
                <span className={`font-medium ${getGoalIcon(message.goalType).color}`}>
                  🎯 {getGoalIcon(message.goalType).label} created!
                </span>
              </div>
              <p className="mt-1 text-gray-600">
                Check the AI Module tab to track your progress. It updates automatically!
              </p>
            </div>
          )}
          // ... rest stays the same
```

---

# 🧪 Testing Your Implementation

## Test Cases to Verify Examples (check with something that makes sense based on the values of your demo)

### Savings Goal
```
User: "I want to save $10000 for a vacation by December 2025 in my Hundred Checking account"
Expected: Creates a savings goal with target $5000 and deadline
```

---

# 🎓 Summary: What You Learned

| Concept | Where Applied |
|---------|---------------|
| **TypeScript Interfaces** | Defining data shapes for type safety |
| **Data Access Layer** | Separating database queries from business logic |
| **LangChain Tools** | Teaching AI agents to perform actions |
| **React Component Patterns** | Conditional rendering, composition |
| **User Experience** | Feedback, celebrations, warnings |
| **Full-Stack Integration** | Connecting frontend → backend → database |

---

# 🚀 Extension Ideas

Want to keep building? Try adding:

1. **Goal Notifications**: Alert users when they're close to a deadline
2. **Goal History**: Track progress over time with a mini chart
3. **Shared Goals**: Let users collaborate on family savings goals
4. **Goal Templates**: Pre-built goals like "Emergency Fund (3 months expenses)"
5. **AI Insights**: "At your current savings rate, you'll reach your goal in X months"
