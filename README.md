# Task
import json
import os
from datetime import datetime, timedelta
from typing import List, Dict, Optional
import re

class Task:
    def __init__(self, id: int, title: str, description: str = "", priority: str = "medium", 
                 due_date: Optional[str] = None, category: str = "general", completed: bool = False):
        self.id = id
        self.title = title
        self.description = description
        self.priority = priority.lower()
        self.due_date = due_date
        self.category = category.lower()
        self.completed = completed
        self.created_at = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    
    def to_dict(self):
        return {
            'id': self.id,
            'title': self.title,
            'description': self.description,
            'priority': self.priority,
            'due_date': self.due_date,
            'category': self.category,
            'completed': self.completed,
            'created_at': self.created_at
        }
    
    @classmethod
    def from_dict(cls, data):
        task = cls(
            id=data['id'],
            title=data['title'],
            description=data.get('description', ''),
            priority=data.get('priority', 'medium'),
            due_date=data.get('due_date'),
            category=data.get('category', 'general'),
            completed=data.get('completed', False)
        )
        task.created_at = data.get('created_at', datetime.now().strftime("%Y-%m-%d %H:%M:%S"))
        return task

class TaskManager:
    def __init__(self, data_file: str = "tasks.json"):
        self.data_file = data_file
        self.tasks: List[Task] = []
        self.next_id = 1
        self.load_tasks()
    
    def load_tasks(self):
        """Load tasks from JSON file"""
        if os.path.exists(self.data_file):
            try:
                with open(self.data_file, 'r') as f:
                    data = json.load(f)
                    self.tasks = [Task.from_dict(task_data) for task_data in data.get('tasks', [])]
                    self.next_id = data.get('next_id', 1)
            except (json.JSONDecodeError, KeyError):
                print("Warning: Could not load existing tasks. Starting fresh.")
                self.tasks = []
                self.next_id = 1
    
    def save_tasks(self):
        """Save tasks to JSON file"""
        data = {
            'tasks': [task.to_dict() for task in self.tasks],
            'next_id': self.next_id
        }
        with open(self.data_file, 'w') as f:
            json.dump(data, f, indent=2)
    
    def add_task(self, title: str, description: str = "", priority: str = "medium", 
                due_date: Optional[str] = None, category: str = "general") -> Task:
        """Add a new task"""
        if not title.strip():
            raise ValueError("Task title cannot be empty")
        
        if priority.lower() not in ['low', 'medium', 'high', 'urgent']:
            priority = 'medium'
        
        task = Task(
            id=self.next_id,
            title=title.strip(),
            description=description.strip(),
            priority=priority.lower(),
            due_date=due_date,
            category=category.strip().lower()
        )
        
        self.tasks.append(task)
        self.next_id += 1
        self.save_tasks()
        return task
    
    def remove_task(self, task_id: int) -> bool:
        """Remove a task by ID"""
        for i, task in enumerate(self.tasks):
            if task.id == task_id:
                del self.tasks[i]
                self.save_tasks()
                return True
        return False
    
    def get_task(self, task_id: int) -> Optional[Task]:
        """Get a task by ID"""
        for task in self.tasks:
            if task.id == task_id:
                return task
        return None
    
    def complete_task(self, task_id: int) -> bool:
        """Mark a task as completed"""
        task = self.get_task(task_id)
        if task:
            task.completed = True
            self.save_tasks()
            return True
        return False
    
    def update_priority(self, task_id: int, new_priority: str) -> bool:
        """Update task priority"""
        if new_priority.lower() not in ['low', 'medium', 'high', 'urgent']:
            return False
        
        task = self.get_task(task_id)
        if task:
            task.priority = new_priority.lower()
            self.save_tasks()
            return True
        return False
    
    def list_tasks(self, filter_completed: bool = False, category: Optional[str] = None, 
                  priority: Optional[str] = None) -> List[Task]:
        """List tasks with optional filters"""
        filtered_tasks = self.tasks
        
        if filter_completed:
            filtered_tasks = [task for task in filtered_tasks if not task.completed]
        
        if category:
            filtered_tasks = [task for task in filtered_tasks if task.category == category.lower()]
        
        if priority:
            filtered_tasks = [task for task in filtered_tasks if task.priority == priority.lower()]
        
        return self.sort_by_priority(filtered_tasks)
    
    def sort_by_priority(self, tasks: List[Task]) -> List[Task]:
        """Sort tasks by priority (urgent > high > medium > low)"""
        priority_order = {'urgent': 4, 'high': 3, 'medium': 2, 'low': 1}
        return sorted(tasks, key=lambda task: priority_order.get(task.priority, 0), reverse=True)
    
    def get_recommendations(self, limit: int = 5) -> List[Dict]:
        """Get task recommendations based on various factors"""
        recommendations = []
        incomplete_tasks = [task for task in self.tasks if not task.completed]
        
        if not incomplete_tasks:
            return [{"type": "info", "message": "Great job! No pending tasks found."}]
        
        # High priority tasks
        urgent_high_tasks = [task for task in incomplete_tasks if task.priority in ['urgent', 'high']]
        if urgent_high_tasks:
            recommendations.append({
                "type": "priority",
                "message": f"You have {len(urgent_high_tasks)} high priority task(s) to focus on:",
                "tasks": urgent_high_tasks[:3]
            })
        
        # Overdue tasks (if due dates are set)
        overdue_tasks = []
        today = datetime.now().date()
        for task in incomplete_tasks:
            if task.due_date:
                try:
                    due_date = datetime.strptime(task.due_date, "%Y-%m-%d").date()
                    if due_date < today:
                        overdue_tasks.append(task)
                except ValueError:
                    continue
        
        if overdue_tasks:
            recommendations.append({
                "type": "overdue",
                "message": f"You have {len(overdue_tasks)} overdue task(s):",
                "tasks": overdue_tasks[:3]
            })
        
        # Category-based recommendations
        category_counts = {}
        for task in incomplete_tasks:
            category_counts[task.category] = category_counts.get(task.category, 0) + 1
        
        if category_counts:
            most_common_category = max(category_counts, key=category_counts.get)
            if category_counts[most_common_category] > 1:
                category_tasks = [task for task in incomplete_tasks if task.category == most_common_category]
                recommendations.append({
                    "type": "category",
                    "message": f"Focus on '{most_common_category}' tasks ({len(category_tasks)} pending):",
                    "tasks": category_tasks[:3]
                })
        
        # Quick wins (tasks with short descriptions, indicating they might be simple)
        quick_tasks = [task for task in incomplete_tasks 
                      if len(task.description) < 50 and task.priority in ['low', 'medium']]
        if quick_tasks:
            recommendations.append({
                "type": "quick_wins",
                "message": f"Quick wins - {len(quick_tasks)} potentially easy task(s):",
                "tasks": quick_tasks[:3]
            })
        
        # Productivity tip
        if len(incomplete_tasks) > 10:
            recommendations.append({
                "type": "tip",
                "message": "Tip: You have many pending tasks. Consider breaking down complex tasks into smaller ones."
            })
        
        return recommendations[:limit]
    
    def search_tasks(self, query: str) -> List[Task]:
        """Search tasks by title or description"""
        query = query.lower()
        results = []
        for task in self.tasks:
            if (query in task.title.lower() or 
                query in task.description.lower() or 
                query in task.category.lower()):
                results.append(task)
        return results
    
    def get_statistics(self) -> Dict:
        """Get task statistics"""
        total_tasks = len(self.tasks)
        completed_tasks = len([task for task in self.tasks if task.completed])
        pending_tasks = total_tasks - completed_tasks
        
        priority_counts = {'low': 0, 'medium': 0, 'high': 0, 'urgent': 0}
        category_counts = {}
        
        for task in self.tasks:
            if not task.completed:
                priority_counts[task.priority] = priority_counts.get(task.priority, 0) + 1
                category_counts[task.category] = category_counts.get(task.category, 0) + 1
        
        return {
            'total_tasks': total_tasks,
            'completed_tasks': completed_tasks,
            'pending_tasks': pending_tasks,
            'completion_rate': round((completed_tasks / total_tasks * 100) if total_tasks > 0 else 0, 1),
            'priority_distribution': priority_counts,
            'category_distribution': category_counts
        }

class TaskManagerCLI:
    def __init__(self):
        self.manager = TaskManager()
        self.commands = {
            '1': self.add_task,
            '2': self.list_tasks,
            '3': self.remove_task,
            '4': self.complete_task,
            '5': self.update_priority,
            '6': self.get_recommendations,
            '7': self.search_tasks,
            '8': self.show_statistics,
            '9': self.show_help,
            '0': self.exit_app
        }
    
    def display_menu(self):
        print("\n" + "="*50)
        print("           TASK MANAGEMENT SYSTEM")
        print("="*50)
        print("1. Add Task")
        print("2. List Tasks")
        print("3. Remove Task")
        print("4. Complete Task")
        print("5. Update Priority")
        print("6. Get Recommendations")
        print("7. Search Tasks")
        print("8. Show Statistics")
        print("9. Help")
        print("0. Exit")
        print("="*50)
    
    def add_task(self):
        print("\n--- Add New Task ---")
        title = input("Task title: ").strip()
        if not title:
            print("Error: Task title cannot be empty!")
            return
        
        description = input("Description (optional): ").strip()
        
        print("Priority levels: low, medium, high, urgent")
        priority = input("Priority (default: medium): ").strip() or "medium"
        
        due_date = input("Due date (YYYY-MM-DD, optional): ").strip()
        if due_date:
            try:
                datetime.strptime(due_date, "%Y-%m-%d")
            except ValueError:
                print("Invalid date format. Due date not set.")
                due_date = None
        
        category = input("Category (default: general): ").strip() or "general"
        
        try:
            task = self.manager.add_task(title, description, priority, due_date, category)
            print(f"✓ Task '{task.title}' added successfully! (ID: {task.id})")
        except ValueError as e:
            print(f"Error: {e}")
    
    def list_tasks(self):
        print("\n--- Task List ---")
        print("Filter options:")
        print("1. All tasks")
        print("2. Pending tasks only")
        print("3. By category")
        print("4. By priority")
        
        choice = input("Choose filter (1-4): ").strip()
        
        tasks = []
        if choice == "1":
            tasks = self.manager.list_tasks()
        elif choice == "2":
            tasks = self.manager.list_tasks(filter_completed=True)
        elif choice == "3":
            category = input("Enter category: ").strip()
            tasks = self.manager.list_tasks(category=category)
        elif choice == "4":
            priority = input("Enter priority (low/medium/high/urgent): ").strip()
            tasks = self.manager.list_tasks(priority=priority)
        else:
            tasks = self.manager.list_tasks(filter_completed=True)
        
        if not tasks:
            print("No tasks found!")
            return
        
        self.display_tasks(tasks)
    
    def display_tasks(self, tasks: List[Task]):
        for task in tasks:
            status = "✓" if task.completed else "○"
            priority_symbol = {"low": "↓", "medium": "→", "high": "↑", "urgent": "‼"}
            
            print(f"\n{status} [{task.id}] {task.title}")
            print(f"   Priority: {priority_symbol.get(task.priority, '→')} {task.priority.upper()}")
            print(f"   Category: {task.category.title()}")
            if task.description:
                print(f"   Description: {task.description}")
            if task.due_date:
                print(f"   Due: {task.due_date}")
            print(f"   Created: {task.created_at}")
    
    def remove_task(self):
        print("\n--- Remove Task ---")
        try:
            task_id = int(input("Enter task ID to remove: "))
            task = self.manager.get_task(task_id)
            if task:
                confirm = input(f"Remove '{task.title}'? (y/N): ").lower()
                if confirm == 'y':
                    self.manager.remove_task(task_id)
                    print("✓ Task removed successfully!")
                else:
                    print("Task removal cancelled.")
            else:
                print("Task not found!")
        except ValueError:
            print("Invalid task ID!")
    
    def complete_task(self):
        print("\n--- Complete Task ---")
        try:
            task_id = int(input("Enter task ID to complete: "))
            task = self.manager.get_task(task_id)
            if task:
                if task.completed:
                    print("Task is already completed!")
                else:
                    self.manager.complete_task(task_id)
                    print(f"✓ Task '{task.title}' marked as completed!")
            else:
                print("Task not found!")
        except ValueError:
            print("Invalid task ID!")
    
    def update_priority(self):
        print("\n--- Update Task Priority ---")
        try:
            task_id = int(input("Enter task ID: "))
            task = self.manager.get_task(task_id)
            if task:
                print(f"Current priority: {task.priority}")
                new_priority = input("New priority (low/medium/high/urgent): ").strip()
                if self.manager.update_priority(task_id, new_priority):
                    print(f"✓ Priority updated to '{new_priority}'!")
                else:
                    print("Invalid priority level!")
            else:
                print("Task not found!")
        except ValueError:
            print("Invalid task ID!")
    
    def get_recommendations(self):
        print("\n--- Task Recommendations ---")
        recommendations = self.manager.get_recommendations()
        
        for rec in recommendations:
            print(f"\n🔍 {rec['message']}")
            if 'tasks' in rec:
                for task in rec['tasks']:
                    priority_symbol = {"low": "↓", "medium": "→", "high": "↑", "urgent": "‼"}
                    print(f"   • [{task.id}] {task.title} ({priority_symbol.get(task.priority, '→')} {task.priority})")
    
    def search_tasks(self):
        print("\n--- Search Tasks ---")
        query = input("Enter search term: ").strip()
        if not query:
            print("Search term cannot be empty!")
            return
        
        results = self.manager.search_tasks(query)
        if results:
            print(f"\nFound {len(results)} task(s):")
            self.display_tasks(results)
        else:
            print("No tasks found matching your search.")
    
    def show_statistics(self):
        print("\n--- Task Statistics ---")
        stats = self.manager.get_statistics()
        
        print(f"Total Tasks: {stats['total_tasks']}")
        print(f"Completed: {stats['completed_tasks']}")
        print(f"Pending: {stats['pending_tasks']}")
        print(f"Completion Rate: {stats['completion_rate']}%")
        
        print("\nPending Tasks by Priority:")
        for priority, count in stats['priority_distribution'].items():
            if count > 0:
                print(f"  {priority.title()}: {count}")
        
        print("\nTasks by Category:")
        for category, count in stats['category_distribution'].items():
            print(f"  {category.title()}: {count}")
    
    def show_help(self):
        print("\n--- Help ---")
        print("This task management app helps you organize your work efficiently.")
        print("\nKey Features:")
        print("• Add tasks with descriptions, priorities, due dates, and categories")
        print("• List and filter tasks by various criteria")
        print("• Set priorities: low, medium, high, urgent")
        print("• Get intelligent recommendations based on your tasks")
        print("• Search tasks by keywords")
        print("• Track completion statistics")
        print("\nTips:")
        print("• Use categories to group related tasks (e.g., 'work', 'personal', 'study')")
        print("• Set due dates for time-sensitive tasks")
        print("• Check recommendations regularly for productivity insights")
        print("• Break large tasks into smaller, manageable ones")
    
    def exit_app(self):
        print("\nThank you for using Task Manager! Your tasks have been saved.")
        return False
    
    def run(self):
        print("Welcome to Task Management System!")
        
        while True:
            self.display_menu()
            choice = input("Choose an option (0-9): ").strip()
            
            if choice in self.commands:
                if choice == '0':
                    if not self.commands[choice]():
                        break
                else:
                    self.commands[choice]()
            else:
                print("Invalid choice! Please select a number from 0-9.")
            
            input("\nPress Enter to continue...")

if __name__ == "__main__":
    app = TaskManagerCLI()
    app.run()
