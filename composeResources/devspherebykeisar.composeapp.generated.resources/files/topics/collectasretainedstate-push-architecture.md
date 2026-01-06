---
id: collectasretainedstate-push-architecture
title: Mastering collectAsRetainedState with Push-Based Data Architecture for Superior UDF
tldr: Using collectAsRetainedState in Circuit presenters combined with push-based data layers creates persistent, reactive UIs that survive configuration changes while maintaining unidirectional data flow and separation of concerns.
tags: [circuit, compose, state-management, architecture, coroutines, flow]
difficulty: advanced
readTimeMinutes: 10
publishedDate: null
relatedTopics: [hot-cold-streams, extension-functions-vs-context-receivers]
---

Traditional Android development often relies on **pull-based** architectures where the UI layer actively requests data. This approach has limitations: unnecessary network calls, complex lifecycle management, and difficulty maintaining state consistency across configuration changes.

Enter the **push-based** architecture with `collectAsRetainedState` - a game-changer that transforms how we handle state in modern Kotlin applications.

### Understanding collectAsRetainedState

`collectAsRetainedState` is a powerful Compose API that collects Flow values as retained State, surviving configuration changes and maintaining UI consistency. When combined with Circuit's presenter pattern, it creates a robust state management system.

#### Basic Example from Tivi

```kotlin
@Composable
fun ShowSeasonsPresenter(
    navigator: Navigator,
    showId: Long,
): ShowSeasonsUiState {
    val scope = rememberCoroutineScope()
    val seasonsRepository = remember { SeasonsRepository() }

    // Push-based: Repository emits updates automatically
    val seasons by seasonsRepository
        .observeSeasonsForShow(showId)
        .collectAsRetainedState(initial = emptyList())

    return ShowSeasonsUiState(
        seasons = seasons,
        eventSink = { event ->
            when (event) {
                is ShowSeasonsEvent.SelectSeason -> {
                    navigator.goTo(SeasonDetails(event.seasonId))
                }
            }
        }
    )
}
```

### The Power of Push-Based Data Layer

A push-based data layer **emits** updates rather than waiting to be queried. This creates a reactive system where changes propagate automatically through your application.

#### Repository Pattern with Flow

```kotlin
class ShowDetailsRepository(
    private val api: TraktApi,
    private val db: ShowDatabase,
    private val dispatcher: CoroutineDispatcher = Dispatchers.IO
) {
    // Push-based: Combines network + database seamlessly
    fun observeShowDetails(showId: Long): Flow<ShowDetails?> =
        combine(
            db.showDao().observeShow(showId),
            db.seasonsDao().observeSeasonsForShow(showId),
            db.episodesDao().observeEpisodesForShow(showId)
        ) { show, seasons, episodes ->
            show?.let {
                ShowDetails(
                    show = it,
                    seasons = seasons,
                    episodes = episodes
                )
            }
        }.flowOn(dispatcher)
        .distinctUntilChanged()
        .onStart {
            // Trigger network refresh in background
            refreshShowDetails(showId)
        }

    private suspend fun refreshShowDetails(showId: Long) {
        withContext(dispatcher) {
            try {
                val details = api.getShowDetails(showId)
                db.withTransaction {
                    db.showDao().upsert(details.show)
                    db.seasonsDao().upsert(details.seasons)
                    db.episodesDao().upsert(details.episodes)
                }
            } catch (e: Exception) {
                // Network errors don't break the Flow
                // Database continues to emit cached data
            }
        }
    }
}
```

### Advanced Presenter with produceState

For more complex scenarios, `produceState` offers fine-grained control over state production:

```kotlin
@Composable
fun ShowDetailsPresenter(
    screen: ShowDetailsScreen,
    navigator: Navigator,
    showRepository: ShowDetailsRepository,
    followedShowsRepository: FollowedShowsRepository,
): ShowDetailsUiState {
    val scope = rememberCoroutineScope()

    // Complex state production with multiple data sources
    val state by produceRetainedState(
        initialValue = ShowDetailsUiState.Loading,
        key1 = screen.showId
    ) {
        // Combine multiple flows for comprehensive state
        combine(
            showRepository.observeShowDetails(screen.showId),
            followedShowsRepository.isShowFollowed(screen.showId),
            showRepository.observeRelatedShows(screen.showId)
        ) { details, isFollowed, relatedShows ->
            when {
                details == null -> ShowDetailsUiState.Loading
                else -> ShowDetailsUiState.Success(
                    show = details.show,
                    seasons = details.seasons,
                    episodes = details.episodes,
                    isFollowed = isFollowed,
                    relatedShows = relatedShows
                )
            }
        }.collect { newState ->
            value = newState
        }
    }

    return state.copy(
        eventSink = { event ->
            when (event) {
                is ShowDetailsEvent.ToggleFollow -> {
                    scope.launch {
                        if ((state as? ShowDetailsUiState.Success)?.isFollowed == true) {
                            followedShowsRepository.unfollowShow(screen.showId)
                        } else {
                            followedShowsRepository.followShow(screen.showId)
                        }
                        // No need to update state manually - Flow emits automatically!
                    }
                }
                is ShowDetailsEvent.Refresh -> {
                    scope.launch {
                        showRepository.refreshShowDetails(screen.showId)
                    }
                }
            }
        }
    )
}
```

### Why This Architecture is Superior

#### 1. True Unidirectional Data Flow (UDF)

Data flows in one direction: Repository → Presenter → UI. Events flow back through the `eventSink`, maintaining clear separation of concerns.

```kotlin
// Data flows down
Repository (Flow) → Presenter (State) → UI (Compose)
                                          ↓
                    EventSink ← Event ← User Interaction
```

#### 2. Automatic State Persistence

`collectAsRetainedState` survives configuration changes without additional boilerplate:

```kotlin
// This survives rotation, no SavedStateHandle needed!
val shows by repository.observeShows()
    .collectAsRetainedState(initial = emptyList())
```

#### 3. Reactive to All Changes

Push-based architecture reacts to changes from any source:

```kotlin
class FollowedShowsRepository(
    private val db: Database,
    private val syncManager: SyncManager
) {
    // Emits updates from local changes OR remote sync
    fun observeFollowedShows(): Flow<List<Show>> =
        merge(
            db.showDao().observeFollowedShows(),
            syncManager.syncUpdates
                .filter { it.type == SyncType.FOLLOWED_SHOWS }
                .flatMapLatest {
                    db.showDao().observeFollowedShows()
                }
        ).distinctUntilChanged()
}
```

#### 4. Optimistic Updates with Fallback

Push-based systems handle optimistic updates elegantly:

```kotlin
class EpisodeRepository(private val db: Database, private val api: Api) {
    fun observeEpisodeWatchedState(episodeId: Long): Flow<Boolean> =
        db.episodeDao().observeWatched(episodeId)
            .distinctUntilChanged()

    suspend fun toggleWatched(episodeId: Long) {
        // Optimistic update - immediately reflected in UI
        val current = db.episodeDao().getWatched(episodeId)
        db.episodeDao().setWatched(episodeId, !current)

        try {
            // Sync with server
            api.updateWatchedState(episodeId, !current)
        } catch (e: Exception) {
            // Revert on failure - UI updates automatically
            db.episodeDao().setWatched(episodeId, current)
            throw e
        }
    }
}
```

#### 5. Efficient Resource Management

Flows are lifecycle-aware and automatically cancelled:

```kotlin
@Composable
fun ShowListPresenter(): ShowListUiState {
    // Flow collection tied to Composition lifecycle
    // Automatically cancelled when Composable leaves composition
    val shows by repository.observeShows()
        .map { shows ->
            shows.sortedByDescending { it.rating }
        }
        .collectAsRetainedState(initial = emptyList())

    // No manual lifecycle management needed!
    return ShowListUiState(shows = shows)
}
```

### Advanced Patterns

#### Combining Multiple Flows with Complex Logic

```kotlin
@Composable
fun HomePresenter(
    trendingRepository: TrendingRepository,
    recommendationEngine: RecommendationEngine,
    watchlistRepository: WatchlistRepository
): HomeUiState {
    val homeContent by produceRetainedState(
        initialValue = HomeContent.Loading
    ) {
        combine(
            trendingRepository.observeTrending(),
            recommendationEngine.observeRecommendations(),
            watchlistRepository.observeUpcoming()
        ) { trending, recommendations, upcoming ->
            HomeContent.Loaded(
                sections = buildList {
                    if (upcoming.isNotEmpty()) {
                        add(HomeSection.ContinueWatching(upcoming))
                    }
                    add(HomeSection.Trending(trending))
                    add(HomeSection.ForYou(recommendations))
                }
            )
        }.catch { error ->
            emit(HomeContent.Error(error.message))
        }.collect { content ->
            value = content
        }
    }

    return HomeUiState(
        content = homeContent,
        eventSink = { /* handle events */ }
    )
}
```

#### Debouncing and Throttling in Push Systems

```kotlin
class SearchRepository {
    private val searchQuery = MutableStateFlow("")

    fun observeSearchResults(): Flow<List<Show>> =
        searchQuery
            .debounce(300) // Wait for user to stop typing
            .distinctUntilChanged()
            .flatMapLatest { query ->
                if (query.isBlank()) {
                    flowOf(emptyList())
                } else {
                    searchShows(query)
                }
            }

    fun updateSearchQuery(query: String) {
        searchQuery.value = query
    }
}
```

### Testing Benefits

Push-based architectures with `collectAsRetainedState` are highly testable:

```kotlin
@Test
fun `presenter emits correct state when repository updates`() = runTest {
    // Given
    val repository = FakeShowRepository()
    val presenter = ShowDetailsPresenter(
        screen = ShowDetailsScreen(showId = 1),
        repository = repository
    )

    // When
    repository.emitShow(testShow)

    // Then - State automatically updates
    val state = presenter.state
    assertThat(state).isInstanceOf(ShowDetailsUiState.Success::class)
    assertThat((state as ShowDetailsUiState.Success).show).isEqualTo(testShow)
}
```

### Conclusion

The combination of `collectAsRetainedState` with push-based data architecture creates a superior development experience:

- **Persistent**: State survives configuration changes automatically
- **Reactive**: UI updates instantly when data changes anywhere
- **Predictable**: True unidirectional data flow
- **Efficient**: Automatic lifecycle management and resource cleanup
- **Testable**: Clear separation of concerns and deterministic behavior

This architecture pattern, exemplified in apps like Tivi, represents the current pinnacle of Android state management - combining the best of Compose, Coroutines, and reactive programming into a cohesive, maintainable solution.