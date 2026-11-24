# Mission Management System

The purpose of this task was to develop a system that allows missions to be added and marked as completed, all while using the existing Inventory HUD.

From this seemingly simple premise, several more complex challenges arose:

- Where to store the missions.
- How to mark them as completed.
- How to properly display them on the HUD.

In the following sections, I will explain how I approached each of these challenges and how I developed a solution.

# Storage

Since missions need to persist throughout the game, they were stored as part of the Game Save Data. However, we also needed a way to track when missions were added or completed.

For this reason, two `FGameplayTagContainer` variables were created to handle mission storage:

```cpp
FGameplayTagContainer ActiveTasks;
FGameplayTagContainer DoneTasks;
```

The Gameplay Tags stored in these containers were used to identify their corresponding entries in the Data Tables, allowing us to retrieve the appropriate mission descriptions.

It is also important to explain the naming convention used for missions. We established the root `Task.` for every mission, followed by a parent-child hierarchy that allowed us to create subtasks.

If a task has no children, it is considered an independent mission. If a task has a parent, it is considered a subtask.

For example:

```text
Task.Father
    Task.Father.Son1
    Task.Father.Son2
```

# Flow

Once we had the containers to store active and completed missions, we needed a way to notify the game whenever a mission was added or completed.

For this purpose, we used Unreal Engine's Gameplay Message Subsystem. By combining this subsystem with the Broadcast Message functionality previously developed for its use in Hollow Sense, I implemented custom listeners capable of detecting specific messages and retrieving the Gameplay Tags broadcast alongside them.

With the messaging system established, the next step was to develop the logic responsible for marking tasks as completed.

At first glance, it might seem sufficient to add missions to `ActiveTasks` and, once completed, remove them from that container and add them to `DoneTasks`.

However, we also needed a way to keep track of completed subtasks and determine whether their parent task could be marked as completed as well.

To achieve this, I implemented the following pseudo-code for a function that would be called whenever a new task was marked as completed, checking whether its parent could also be completed:

### Pseudo-Code

```text
FUNCTION F1(DoneTask)

    // Check if the completed task has children
    ChildTags = GetChildrenTags(DoneTask)

    IF ChildTags is NOT EMPTY THEN
        PRINT "Task is not a subtask"
        RETURN
    END IF

    // Get the direct parent of the completed task
    ParentTag = GetDirectParent(DoneTask)

    IF ParentTag is NOT VALID THEN
        PRINT "Parent task is not valid"
        RETURN
    END IF

    // Get all subtasks belonging to the parent task
    Parent_ChildTags = GetChildrenTags(ParentTag)

    AllChildrenDone = TRUE

    // Check whether all subtasks have been completed
    FOR EACH ChildTag IN Parent_ChildTags DO

        IF ChildTag is NOT IN DoneTasks THEN
            AllChildrenDone = FALSE
            BREAK
        END IF

    END FOR

    // Complete the parent task if all subtasks are done
    IF AllChildrenDone THEN

        REMOVE ParentTag FROM ActiveTasks
        ADD ParentTag TO DoneTasks

        PRINT "Parent task completed"

    END IF

END FUNCTION
```

# HUD Integration

Now that we have explained how missions are categorized and managed, we can move on to displaying them through the existing HUD.

First, we added a Vertical Box to display the mission descriptions using Text Blocks.

The idea was to use this Vertical Box as a page, displaying a limited number of missions until it was filled, at which point the remaining missions would be displayed on the next page.

To achieve this, we sorted all existing missions from both `ActiveTasks` and `DoneTasks`. Their order was determined by their corresponding IDs in the Data Table, allowing us to control their display order, completion indicators, and page distribution.

To handle this process, we implemented the following pseudo-code:

### Pseudo-Code

```text
FUNCTION F2()

    // Clear existing task list
    CLEAR VerticalBox_Tasks


    // Reset task maps
    CLEAR Tag_Task_Map
    CLEAR IsDone_Task_Map


    // Process active and completed tasks
    FOR EACH TaskTag IN ActiveTasks AND DoneTasks DO

        // Find the corresponding Data Table entry
        TaskRow = FindTaskInDataTable(TaskTag)

        IF TaskRow EXISTS THEN

            TaskID = GetTaskID(TaskRow)

            // Store task information
            Tag_Task_Map[TaskID] = TaskTag

            // Store completion status
            IsDone_Task_Map[TaskID] = IsTaskCompleted(TaskTag)

        END IF

    END FOR


    // Get and sort task IDs
    Id_Keys = GET KEYS FROM Tag_Task_Map
    SORT Id_Keys IN ASCENDING ORDER


    // Create an array to sort the ID for the last page of every page
    CLEAR Id_Tops_Array

    FOR EACH index in Id_Keys:
      
        Id_Top = index + 1 // Id of the task that will belong to the next page
  
        IF Id_Top is a valid index in Id_Keys:

            IF Id_Top % Tasks_Per_Page == 0: // We found a potential Id_Top, now we will check the heirarchy, to arrange the task with it´s parent

                TaskTag = Tag_Task_Map[Id_Keys[Id_Top]]
                ParentTag = GetDirectParent(TaskTag)

                // We already found a potential Top, we will exclude it if it has a parent, and exluce the parent as well
                WHILE ParentTag IS VALID THEN

                  Id_Top = Id_Top - 1
                  TaskTag = Tag_Task_Map[Id_Keys[Id_Top]]
                  ParentTag = GetDirectParent(TaskTag)

                END WHILE
                ADD Id_Top - 1 to Id_Tops_Array
            ELSE:

                skip

            END ELSE
            END IF
        ELSE:

            ADD Id_Top - 1 to Id_Tops_Array

        END ELSE

        END IF

    END FOR

END FUNCTION
```

Finally, if `Id_Keys` is not empty, we can start displaying the mission descriptions inside the Vertical Box according to the currently selected page (`ap`).

The tasks displayed will correspond to the IDs within the range:

```text
[Id_Tops_Array[ap-1] + 1, Id_Tops_Array[ap]]
```

Where we define:

```text
Id_Tops_Array[-1] = -1
```

This allows us to determine which tasks belong to each page while keeping their hierarchy properly organized.

Finally, we use `IsDone_Task_Map` to determine whether a visual indicator should be added to each task, showing whether it has been completed or is still active.

And, we will use IsDone_Tasks_Map to determinate if we will add a visual clue to mark the task as done or not.
