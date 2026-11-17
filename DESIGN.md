# M5 Design

## M4 Fixes
My M4 implementation and CR-1 behavior did not require any fixes before starting M5.

## Threads and Shared Data
The GUI thread owns MainWindow, the widgets, workspace_, processing_core_, and corpus_state_. One persistent worker thread runs run_worker() from MainWindow construction until exit. The GUI copies the Workspace, generation, build delay, and promise into the pending request. build_mutex_ protects pending_workspace_, pending_generation_, pending_delay_ms_, pending_promise_, and build_requested_. stop_worker_ and cancel_build_ are atomic because both the GUI and worker threads access them. The worker waits on build_cv_ instead of polling and builds from a local Workspace copy. Results return through the promise and build_finished() is queued to the GUI thread. State objects stay on the GUI thread. The CR-1 listener queues remote adds to the GUI thread.

## State Pattern
EmptyState: no documents and no current corpus.  
StaleState: documents exist, but the corpus is not current.  
BuildingState: the worker is handling a corpus build.  
CurrentState: the corpus matches the Workspace.

Transitions:

Empty + Add/remote -> Stale  
Stale + Build -> Building  
Current + Build -> Building  
Current + Add/remote -> Stale  
Stale/Current + Reset -> Empty  
Building + Add/remote -> stay Building, cancel the obsolete build, allow one replacement Build  
Building + Build after Workspace change -> stay Building and remember the replacement  
Building + Reset -> stay Building while cancelling, then Empty  
Building + obsolete completion with replacement requested -> stay Building and start replacement  
Building + successful current build -> Current  
Building + stale/failed build -> Stale or Empty

MainWindow sends lifecycle events to the current State object, which decides the behavior and next State. The old corpus_current_ and build_running_ flags were removed.

## Event Trace
A Workspace change during a long build threatens R2 and R3. add_document_file() updates the Workspace and sends workspace_changed() to BuildingState. BuildingState::workspace_changed() sets cancel_build_, notifies build_cv_, invalidates the corpus, and allows one replacement Build. If Build is pressed, BuildingState::build_requested() remembers it. The worker wakes, sees cancel_build_, stops the obsolete build, and queues build_finished(). BuildingState::build_finished() then calls start_build() for the remembered request. The old build never becomes current, and the replacement does not wait for all of its remaining document delays.

## Design Decisions
Build is disabled while a useful build is running. After a Workspace change makes it obsolete, BuildingState allows one replacement Build request and remembers it until cancellation finishes. A remote add uses add_document_file(), so it cancels the running build and makes the corpus stale just like Add Document. The only shared cross-thread data is the pending request, build_requested_, stop_worker_, and cancel_build_; these are needed for handoff, shutdown, and cancellation, and are protected by build_mutex_ or atomics.

## Fault Switches and Tests
AIWS_FAULT_NO_CANCEL is in run_worker() around the cancellation checks; cancels_outdated_build() catches it. AIWS_FAULT_SILENT_SHUTDOWN is in MainWindow::~MainWindow(); idle_shutdown() catches it. state_lifecycle() checks the main State sequence. Normal tests pass, and the targeted ThreadSanitizer run reported no data race.

## AI Use
I used AI to review the requirements, discuss design choices, and help debug my M5 implementation and tests. I reviewed, edited, built, and tested the submitted code myself.
