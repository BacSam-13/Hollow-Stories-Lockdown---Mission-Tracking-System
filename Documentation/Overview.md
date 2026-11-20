The porpouse of this task, was to develop a system to add missions, and being able to mark them as completed. All this, using the already existing inventory HUD.
From that simple premise, new complex issues came aboard:
  - Where to store the missions
  - How mark them as completed
  - How to deploy them properly on the HUD

Next, I will explain how I approached all of these issues, and how I came to the solution.

# Storage
Since the missions will be used during all game, they were stored as part of the GameSave Data, but we also needed a way to being able to listen when the missions were added and completed.
That´s why two FGameplayTagContainer were created, to help storage the data:
	FGameplayTagContainer ActiveTasks;
	FGameplayTagContainer DoneTasks;
The GameplayTags used on these containers, were used as keys on Data Tables, so we could get acces to the proper text of the mission.
It is also important to mention the nomenclature used for the missions. We used the root "Task." for every task, and next, we used a system of parents and childs, so we could use
sub-missions. In case a task has no child, it is considered a single mission, if a task has a father, it is a submission. For example:
  Task.Father
    Task.Father.Son1
    Task.Father.Son2


# Flow
Once we already had the containers to storage the Active and Done missions, we needed a way to being able to tell the game when were the missions added, or completed.
That´s when we used the Gameplay Message subsystem of Unreal Enngine. Using this subsystem, alonside to the Broadcast Message developed for it´s use in Hollow Sense, I wrote
some special listeners, that would listen to the Gameplay Tags broadcasted alonside an specific Message.

Once we defined the Message subsystem, it was time to develop the logic inside marking the tasks as Done, or not.
At first sight, it would be easy to assume we only hace to add the tasks first to ActiveTasks, and once the are completed, we remove them from there, and add it to DoneTasks.
But we needed a proper way to take track of the child tasks that were completed, and in case the were completed, mark the father as completed as well.
This is were I implemented the next Pseudo-Code for a function that will be called each time a new task is marked as Done, to see if the parent could be Done as well:

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

    // Complete the parent task if all its subtasks are done
    IF AllChildrenDone THEN

        REMOVE ParentTag FROM ActiveTasks
        ADD ParentTag TO DoneTasks

        PRINT "Parent task completed"

    END IF

END FUNCTION
```

# Displayment
We have already explained how we categorize the tasks, now, we need to explain how to display them on the already existing HUD.

First of all, we placed a Vertical Box, so we could display Text Blocks on it.
The idea was to use this Vertical Box as a "page", and display some tasks on it, until it is filled, and starts filling the next one.
For this, we would sort all the existing tasks, in both ActiveTasks and DoneTaks, and using the order in the Data Table to decide in which order will the missions be stocked,
which ones will have a visual clue to indicate they were completed, and in which page they will be displayed.
For this, we implemented the next Pseudo-Code:

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

Finally, if Id_Keys is not empty, we can start placing the tasks´ texts in the Vertical Box, according to the actual page (ap),
displaying the tasks from the ID´s [ Id_Tops_Array[ap-1]+1, Id_Tops_Array[ap]], where we define Id_Tops_Array[-1] = -1.

And, we will use IsDone_Tasks_Map to determinate if we will add a visual clue to mark the task as done or not.
