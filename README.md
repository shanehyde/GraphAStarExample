# Hexagonal grid pathfinding in Unreal Engine 4

## Prerequisites
* [A good knowledge of C++.](https://www.learncpp.com/)
* Red Blob Games articles about hexagonal grid.
  * [Hexagonal grid reference.](https://www.redblobgames.com/grids/hexagons/)
  * [Hexagonal grid implementation guide.](https://www.redblobgames.com/grids/hexagons/implementation.html)
  
## These readings will let you better understand my example project
* [Epic wiki article about replacing the pathfinder.](https://wiki.unrealengine.com/Replacing_The_Pathfinder)
* [Neatly replacing NavMesh with A* in UE4 by Chris Russel.](https://crussel.net/2016/06/05/neatly-replacing-navmesh-with-a-in-ue4/)
* [Epic wiki article about custom Path Following Component.](https://wiki.unrealengine.com/AI_Navigation_in_C%2B%2B,_Customize_Path_Following_Every_Tick)

## The GraphAStarExample project
Welcome all to my example project about how to use the Unreal Engine 4 generic graph A* implementation with hexagonal grids, the intent of this project is to guide you on what you need to setup the basics for navigation on hexagonal grids, this is not a complete tutorial but more like a guideline and doesn't cover topics like avoidance, grid transfer etc.

In the project you will find two examples, A and B, example A use the default AIController (that come with the default PathFollowingComponent), the example B use a custom AIController (BP_AIController_Example_B) with a custom PathFollowingComponent (HGPathFollowingComponent).

Both examples work with the same Pathfinder, Example B is just a bonus that will show you what a custom PathFollowingComponent can do.

### Classes and Structs you need to know
Let me start with a list of the classes you have to know and explore in the engine
1. ANavigationData
2. ARecastNavMesh
3. FGraphAStar
3. AAIController (optional)
4. UPathFollowingComponent (optional)

#### ANavigationData
Represents abstract Navigation Data (sub-classed as NavMesh, NavGraph, etc).
Used as a common interface for all navigation types handled by NavigationSystem.

Here you will find a lot of interesting stuff but the most important for us is the FindPathImplementation class member, this is a function pointer defined as FFindPathPtr (typedef).
```
typedef FPathFindingResult (*FFindPathPtr)(const FNavAgentProperties& AgentProperties, const FPathFindingQuery& Query);
FFindPathPtr FindPathImplementation;
```
So the ANavigationData::FindPath function call this function pointer that will be in charge of run the operations.
```
/** 
 *	Synchronously looks for a path from @StartLocation to @EndLocation for agent with properties @AgentProperties. 
 *  NavMesh actor appropriate for specified FNavAgentProperties will be found automatically
 *	@param ResultPath results are put here
 *	@return true if path has been found, false otherwise
 *
 *	@note don't make this function virtual! Look at implementation details and its comments for more info.
 */
FORCEINLINE FPathFindingResult FindPath(const FNavAgentProperties& AgentProperties, const FPathFindingQuery& Query) const
{
	  check(FindPathImplementation);
		// this awkward implementation avoids virtual call overhead - it's possible this function will be called a lot
		return (*FindPathImplementation)(AgentProperties, Query);
}
```
Take a look at the note "don't make this function virtual!", we will come back on it in a while.

#### ARecastNavMesh
This class inherit from ANavigationData and extend his functionality, everytime you place a NavMeshBoundsVolume in the map an object of this class is created in the map, yes, is the RecastNavMesh-Default object!

This is the class we have to inherit from!

Let's look how this class implement the ANavigationData::FindPath function, we already know that this function is not virtual, so how we can implement it? We have the FindPathImplementation function pointer!

In the header you can see the ARecastNavMesh::FindPath declaration.

`static FPathFindingResult FindPath(const FNavAgentProperties& AgentProperties, const FPathFindingQuery& Query);`

the function is static for a reason, (wiki copy-paste->) *comments in the code explain it’s for performance reasons: Epic are concerned that if a lot of agents call the pathfinder in the same frame the virtual call overhead will accumulate and take too long, so instead the function is declared static and stored in the FindPathImplementation function pointer.
Which means you need to manually set the function pointer in your new navigation class constructor* (or in some other function like i did in my example). 

This is the function where we will implement (and pass to the FindPathImplementation pointer) in our inherited class!

#### FGraphAStar
Finally we are in the core class (ok ok, is a struct) of our example, the FGraphAstar is the Unreal Engine 4 generic implementation of the A* algorithm.

If you open the GraphAStar.h file in UE4 you will find in the comments an explanation on how to use it, let's look:

> Generic graph A* implementation.

TGraph holds graph representation. Needs to implement functions:

```
/* Returns number of neighbors that the graph node identified with NodeRef has */
int32 GetNeighbourCount(FNodeRef NodeRef) const;

/* Returns whether given node identification is correct */
bool IsValidRef(FNodeRef NodeRef) const;

/* Returns neighbor ref */
FNodeRef GetNeighbour(const FNodeRef NodeRef, const int32 NeighbourIndex) const;
```
it also needs to specify node type

`FNodeRef - type used as identification of nodes in the graph`

> TQueryFilter (FindPath's parameter) filter class is what decides which graph edges can be used and at what cost. It needs to implement following functions:

```
/**
* Used as GetHeuristicCost's multiplier
 */
float GetHeuristicScale() const;

/**
 * Estimate of cost from StartNodeRef to EndNodeRef
 */
float GetHeuristicCost(const int32 StartNodeRef, const int32 EndNodeRef) const;

/**
 * Real cost of traveling from StartNodeRef directly to EndNodeRef
 */
float GetTraversalCost(const int32 StartNodeRef, const int32 EndNodeRef) const;

/**
 * Whether traversing given edge is allowed
 */
bool IsTraversalAllowed(const int32 NodeA, const int32 NodeB) const;

/**
 * Whether to accept solutions that do not reach the goal
 */
bool WantsPartialSolution() const;
```

So we don't have to create a class (ok ok, is a struct) from FGrapAStar but we have to implement the above functions, typedefs and structs in the class that will be in charge of run the pathfinder.

In the engine code there is a good example on how to do it, go and look the NavLocalGridData.h file.
In our example we will implement the code requested by FGraphAStar in our ARecastNavMesh inherited class, we will talk about it in a moment.

For now this is all for the foundamental class we need, at the end the only class we really will use is the ARecastNavMesh class.

#### AAIController (optional)
In the project you will find two examples, A and B, example A use the default AIController (that come with the default PathFollowingComponent), the example B use a custom AIController (BP_AIController_Example_B) with a custom PathFollowingComponent (HGPathFollowingComponent).

To make the custom PathFollowingComponent work we have to inherit the AAIController class and tell her which class of the PathFollowingComponent we want to use.

We will talk about it later , in the AHGAIController section.

#### UPathFollowingComponent (optional)
This component is in charge to let your AI follow the path, it's full of interesting functions and members, we will override only two of these functions just to show you they are here and what you can do with a custom PathFollowingComponent.

We will talk about it later , in the UHGPathFollowingComponent section.

### Classes and structs we used in the project
1. AGraphAStarNavMesh (inherited from ARecastNavMesh)
2. AHexGrid (inherited from AActor)
3. HGTypes (a collection of structs used by AHexGrid)
4. AHGAIController (inherited from AAIController)
4. UHGPathFollowingComponent (inherited from UPathFollowingComponent)

#### AGraphAStarNavMesh
Our most important class, where the magic happen!

Here is where we "integrate" the FGraphAStar implementation and it will be very easy!

##### FGridPathFilter
First thing first, let's start define the FGridPathFilter struct in the header
```
/**
 * TQueryFilter (FindPath's parameter) filter class is what decides which graph edges can be used and at what cost.
 */
struct FGridPathFilter
{
	FGridPathFilter(const AGraphAStarNavMesh &InNavMeshRef) : NavMeshRef(InNavMeshRef) {}

	/**
	 * Used as GetHeuristicCost's multiplier
	 */
	float GetHeuristicScale() const;

	/**
	 * Estimate of cost from StartNodeRef to EndNodeRef
	 */
	float GetHeuristicCost(const int32 StartNodeRef, const int32 EndNodeRef) const;

	/**
	 * Real cost of traveling from StartNodeRef directly to EndNodeRef
	 */
	float GetTraversalCost(const int32 StartNodeRef, const int32 EndNodeRef) const;

	/**
	 * Whether traversing given edge is allowed
	 */
	bool IsTraversalAllowed(const int32 NodeA, const int32 NodeB) const;

	/**
	 * Whether to accept solutions that do not reach the goal
	 */
	bool WantsPartialSolution() const;

protected:

	/**
	 * A reference to our NavMesh
	 */
	const AGraphAStarNavMesh &NavMeshRef;
};
```

as you see it this struct also has a reference to our AGraphAStarNavMesh, we will use it in the implementation

```
float FGridPathFilter::GetHeuristicScale() const
{
	// For the sake of simplicity we just return 1.f
	return 1.0f;
}

float FGridPathFilter::GetHeuristicCost(const int32 StartNodeRef, const int32 EndNodeRef) const
{
	return GetTraversalCost(StartNodeRef, EndNodeRef);
}

float FGridPathFilter::GetTraversalCost(const int32 StartNodeRef, const int32 EndNodeRef) const
{
	// If EndNodeRef is a valid index of the GridTiles array we return the tile cost, 
	// if not we return 1 because the traversal cost need to be > 0 or the FGraphAStar will stop the execution
	// look at GraphAStar.h line 244: ensure(NewTraversalCost > 0);
	if (NavMeshRef.HexGrid->GridTiles.IsValidIndex(EndNodeRef))
	{
		return NavMeshRef.HexGrid->GridTiles[EndNodeRef].Cost;
	}
	else
	{
		return 1.f;
	}
}

bool FGridPathFilter::IsTraversalAllowed(const int32 NodeA, const int32 NodeB) const
{
	// If NodeB is a valid index of the GridTiles array we return bIsBlocking, 
	// if not we assume we can traverse so we return false.
	// Here you can make a more complex operation like use a line trace to see
	// there is some obstacles (like an enemy), in our example we just use a simple implementation
	if (NavMeshRef.HexGrid->GridTiles.IsValidIndex(NodeB))
	{
		return !NavMeshRef.HexGrid->GridTiles[NodeB].bIsBlocking;
	}
	else
	{
		return true;
	}
	
}

bool FGridPathFilter::WantsPartialSolution() const
{
	// Just return true
	return true;
}
```

##### FGraphAStar functions and typedef
# ARRIVATO QUI

#### AHexGrid

#### HGTypes

#### AHGAIController (optional)
We continue discuss about why we have to inherit this class if we want to use a custom PathFollowingComponent.

We need to tell to the inherited AAIController class which class of the PathFollowingComponent we want to use, to do it we need the ObjectInitializer base class member and the SetDefaultSubobjectClass function and call it when we initialize the base class in the derived class constructor... very easy right? 

Ok ok, here i'm failing hard with my english (i'm very sorry) but don't worry, it is easier said than done:

```
AHGAIController::AHGAIController(const FObjectInitializer &ObjectInitializer /*= FObjectInitializer::Get()*/)
	: Super(ObjectInitializer.SetDefaultSubobjectClass<UHGPathFollowingComponent>(TEXT("PathFollowingComponent")))
{
}
```

that's all, in the contructor we use `: Super()` and we pass the ObjectInitializer (and the SetDefaultSubobjectClass function call) to the parent class constructor, this will replace the default PathFollowingComponent class with ours.

So now when we create an AIController Blueprint based to our class instead of have the default PathFollowingComponent it will have our version but keep attention to the TEXT() parameter, to work well it must be the same as the default component of the parent AIController class! ("PathFollowingComponent")

In your Blueprint you will still see the component named PathFollowingComponent but if you go over it with the mouse you will see it is our derived PathFollowingComponent. (so magic)
# immagine qui

**NOTE:** There is a SetPathFollowingComponet function in the AAIController (also is BlueprintCallable) but i still have to figure out how it work, that's why i preferred the ObjectInitializer method.

#### UHGPathFollowingComponent (optional)
With this class i want to show you how powerfull this component can be, in our example we override two function, we will do something very simple.

The first function we are overriding is OnActorBump:

```
/** called when moving agent collides with another actor */
virtual void OnActorBump(AActor *SelfActor, AActor *OtherActor, FVector NormalImpulse, const FHitResult &Hit) override;
```

this function is called when the Pawn possesed by the AIController bump against another actor (it's like a BeginOverlap, or a Hit, you already know this kind of behavior).

Instead of expose this function to Blueprint i decided to create a delegate so we can bind it where we want.

```
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnActorBumpDelegate, const FVector&, BumpLocation);
.
.
.
/**
 * Executed if a "Bump" happen, we bind this delegate on the activation of our Behavior Tree Service BTS_BindBump
 */
UPROPERTY(BlueprintAssignable, Category = "GraphAStarExample|PathFollowingComponent")
FOnActorBumpDelegate OnActorBumped;
```

In our example we will bind it in a Behavior Tree Service (see how in the BTS_BindBump section).

Now the OnActorBump implementation where we look if the bump happen when we are moving or waiting, if yes we Broadcast the location of the other actor involved in the bump.

```
void UHGPathFollowingComponent::OnActorBump(AActor *SelfActor, AActor *OtherActor, FVector NormalImpulse, const FHitResult &Hit)
{
	Super::OnActorBump(SelfActor, OtherActor, NormalImpulse, Hit);

	// Let's see if we are moving or waiting.
	if (GetStatus() != EPathFollowingStatus::Idle)
	{
		// Just broadcast the event.
		OnActorBumped.Broadcast(OtherActor->GetActorLocation());
	}
}
```

The second function i decided to override is one of the most imnportant we can find in the PathFollowingComponent, the FollowPathSegment function, is the main path follow function and it tick while we are traveling the path!

We will use it for simple debug purposes, we draw the current followed path and the start point and the end point of the current path segment we are traveling!

```
/** follow current path segment */
virtual void FollowPathSegment(float DeltaTime) override;

void UHGPathFollowingComponent::FollowPathSegment(float DeltaTime)
{
	Super::FollowPathSegment(DeltaTime);

	/**
	 * FollowPathSegment is the main UE4 Path Follow tick function, and so when you want to add completely 
	 * custom coding you can use this function as your starting point to adjust normal UE4 path behavior!
	 *
	 * Let me show you a simple example with some debug drawings.
	 */

	if (Path && DrawDebug)
	{
		// Just draw the current path
		Path->DebugDraw(MyNavData, FColor::White, nullptr, false);
		
		// Draw the start point of the current path segment we are traveling.
		FNavPathPoint CurrentPathPoint{};
		FNavigationPath::GetPathPoint(&Path->AsShared().Get(), GetCurrentPathIndex(), CurrentPathPoint);
		DrawDebugLine(GetWorld(), CurrentPathPoint.Location, CurrentPathPoint.Location + FVector(0.f, 0.f, 200.f), FColor::Blue);
		DrawDebugSphere(GetWorld(), CurrentPathPoint.Location + FVector(0.f, 0.f, 200.f), 25.f, 16, FColor::Blue);

		// Draw the end point of the current path segment we are traveling.
		FNavPathPoint NextPathPoint{};
		FNavigationPath::GetPathPoint(&Path->AsShared().Get(), GetNextPathIndex(), NextPathPoint);
		DrawDebugLine(GetWorld(), NextPathPoint.Location, NextPathPoint.Location + FVector(0.f, 0.f, 200.f), FColor::Green);
		DrawDebugSphere(GetWorld(), NextPathPoint.Location + FVector(0.f, 0.f, 200.f), 25.f, 16, FColor::Green);
	}
}

```
# immagine qui
