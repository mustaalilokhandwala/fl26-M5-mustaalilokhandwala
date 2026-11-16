# M5 Design

## M4 Fixes
My M4 implementation and CR-1 behavior did not require any fixes before starting M5.

## Threads and Shared Data
The GUI thread handles MainWindow, all of the widgets, the Workspace, the current ProcessingCore, and the current State object. M5 uses one worker thread for every corpus build instead of creating a new thread for each build. The worker starts when MainWindow is created and stays alive until the application exits. When the user starts a build, the GUI thread copies the Workspace, workspace generation, build delay, and promise into the pending build data. build_mutex_ protects pending_workspace_, pending_generation_, pending_delay_ms_, pending_promise_, and build_requested_. stop_worker_ and cancel_build_ are atomic because both the GUI thread and worker thread can use them safely. The worker sleeps on build_cv_ until there is a build to do or the application is closing, so it does not keep checking in a loop. The worker builds from its own local copy of the Workspace. When it finishes, it sends the result through the promise and queues build_finished() back to the GUI thread. The State objects are only used by the GUI thread. The CR-1 listener still runs on its own thread, but remote adds are sent to the GUI thread before changing the Workspace.

## State Pattern

States:

EmptyState  
The Workspace has no documents and there is no current corpus.

StaleState  
The Workspace has documents, but the corpus is not current.

BuildingState  
A corpus build is currently being handled by the worker.

CurrentState  
The corpus matches the current Workspace and can be used.

Transitions:

EmptyState + Add/remote add -> StaleState

StaleState + Build -> BuildingState

CurrentState + Build -> BuildingState

CurrentState + Add/remote add -> StaleState

StaleState + Reset -> EmptyState

CurrentState + Reset -> EmptyState

BuildingState + Add/remote add -> stay in BuildingState and cancel the current build

BuildingState + Reset -> stay in BuildingState while the current build is cancelled, then become EmptyState

BuildingState + successful current build -> CurrentState

BuildingState + stale or failed build -> StaleState or EmptyState

MainWindow sends events such as Add, Build, Reset, Exit, and build completion to the current State object. The State object decides what should happen and which State comes next. The old corpus_current_ and build_running_ flags were removed.

## Event Trace
If the application is in StaleState and the user presses Build, build_corpus() sends the event to StaleState::build_requested(). That State calls start_build() and changes the application to BuildingState. start_build() saves the build request while holding build_mutex_ and wakes the worker. The worker takes the request, builds from its copied Workspace, and checks cancel_build_ while the build is running. When it finishes, it sends the result back to the GUI thread. build_finished() gives the result to BuildingState, which decides whether the next State should be CurrentState, StaleState, or EmptyState.

## Design Decisions
Build stays disabled while another build is running, so the user cannot start a second build at the same time. A remote add goes through add_document_file(), just like a normal Add Document action. If an add happens during a build, the running build is cancelled and the corpus becomes outdated. The two threads only share the pending build information, the build request flag, and the stop and cancel flags.

## Fault Switches and Tests
AIWS_FAULT_NO_CANCEL makes an outdated build continue instead of stopping. cancels_outdated_build() catches this because the old build takes too long to finish. AIWS_FAULT_SILENT_SHUTDOWN prevents an idle worker from stopping, so idle_shutdown() hangs and times out. state_lifecycle() checks the main State sequence from Empty to Stale to Building to Current and back to Empty. The normal test suite passes, and the targeted ThreadSanitizer run completed without reporting a data race.

## AI Use
I used AI as an assistant to review the assignment requirements, discuss design choices, and help debug parts of my M5 implementation and tests. I reviewed, edited, built, and tested the submitted code myself.
