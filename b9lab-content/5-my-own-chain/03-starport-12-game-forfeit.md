---
title: The Expired Game Elements
order: 14
description: You enforce expiration of games.
---

# The Expired Game Elements

In the previous sections you prepared the ground to be able to enforce a expiration games, namely:

* A FIFO that always has old games at its head and freshly updated games at its tail.
* A deadline field to guide the expiration.
* A winner field to further assist with forfeiting.

## New Information

An expired game will expire in two different ways:

1. It was never really played on, so it is removed quietly.
2. It was played on, making a proper game, expiry here means a forfeit by the player who failed to play.

In the second case, you want to emit a new event, one that differentiates it from a win where a move was involved. Let's define the new keys in `x/checkers/types/keys.go`:

```go [https://github.com/cosmos/b9-checkers-academy-draft/blob/7739537804f350241f59ee55b443a66b68883fc8/x/checkers/types/keys.go#L66-L70]
const (
    ForfeitGameEventKey     = "GameForfeited"
    ForfeitGameEventIdValue = "IdValue"
    ForfeitGameEventWinner  = "Winner"
)
```

## Putting Callbacks In Place

When you used Starport to scaffold your module, it created the `x/checkers/module.go` file with a lot of functions to accommodate the larger application. In particular, the function that **may** be called on your module on `EndBlock` is named `EndBlock`:

```go [https://github.com/cosmos/b9-checkers-academy-draft/blob/7739537804f350241f59ee55b443a66b68883fc8/x/checkers/module.go#L163]
func (am AppModule) EndBlock(_ sdk.Context, _ abci.RequestEndBlock) []abci.ValidatorUpdate {
    return []abci.ValidatorUpdate{}
}
```
Starport left it empty, and this is where you add what you need. To keep with the spirit of Starport that separates different concerns into different files, you create a new file named `x/checkers/keeper/end_block_server_game.go` with a function in:

```go [https://github.com/cosmos/b9-checkers-academy-draft/blob/7739537804f350241f59ee55b443a66b68883fc8/x/checkers/keeper/end_block_server_game.go#L13]
func (k Keeper) ForfeitExpiredGames(goCtx context.Context) {
    // TODO
}
```
Such that, in `x/checkers/module.go`, you can update `EndBlock` with:

```go [https://github.com/cosmos/b9-checkers-academy-draft/blob/7739537804f350241f59ee55b443a66b68883fc8/x/checkers/module.go#L163-L166]
func (am AppModule) EndBlock(ctx sdk.Context, _ abci.RequestEndBlock) []abci.ValidatorUpdate {
    am.keeper.ForfeitExpiredGames(sdk.WrapSDKContext(ctx))
    return []abci.ValidatorUpdate{}
}
```
With this, you ensure that **if** your module's `EndBlock` function is called, then expired games will be handled. But for the **whole application to call your module**, you have to instruct it. This takes place in `app/app.go`, where the application is initialized with the proper order in which to call the `EndBlock` functions in different modules. So add yours at the end:

```go [https://github.com/cosmos/b9-checkers-academy-draft/blob/7739537804f350241f59ee55b443a66b68883fc8/app/app.go#L398]
app.mm.SetOrderEndBlockers(crisistypes.ModuleName, govtypes.ModuleName, stakingtypes.ModuleName, checkersmoduletypes.ModuleName)
```

## Expire Games Handler

With the callbacks in place, it is time to code the expiry proper. In `x/checkers/keeper/end_block_server_game.go`, it is just a matter of going through the FIFO head and handle games as long as they are in fact expired. You can stop at the first running game as all those that come after are also running.

* Prepare useful information:
    ```go [https://github.com/cosmos/b9-checkers-academy-draft/blob/7739537804f350241f59ee55b443a66b68883fc8/x/checkers/keeper/end_block_server_game.go#L14-L19]
    ctx := sdk.UnwrapSDKContext(goCtx)

    opponents := map[string]string{
        rules.BLACK_PLAYER.Color: rules.RED_PLAYER.Color,
        rules.RED_PLAYER.Color:   rules.BLACK_PLAYER.Color,
    }
    ```
* Initialize the parameters before entering the loop:
    ```go [https://github.com/cosmos/b9-checkers-academy-draft/blob/7739537804f350241f59ee55b443a66b68883fc8/x/checkers/keeper/end_block_server_game.go#L22-L28]
    nextGame, found := k.GetNextGame(ctx)
    if !found {
        panic("NextGame not found")
    }
    storedGameId := nextGame.FifoHead
    var storedGame types.StoredGame
    ```
* Enter the loop:
    ```go [https://github.com/cosmos/b9-checkers-academy-draft/blob/7739537804f350241f59ee55b443a66b68883fc8/x/checkers/keeper/end_block_server_game.go#L29]
    for {
        // TODO
    }
    ```
    * Then in the loop, start with the loop breaking condition:
        ```go [https://github.com/cosmos/b9-checkers-academy-draft/blob/7739537804f350241f59ee55b443a66b68883fc8/x/checkers/keeper/end_block_server_game.go#L31-L33]
        if strings.Compare(storedGameId, types.NoFifoIdKey) == 0 {
            break
        }
        ```
    * Fetch the expired game candidate and its deadline:
        ```go [https://github.com/cosmos/b9-checkers-academy-draft/blob/7739537804f350241f59ee55b443a66b68883fc8/x/checkers/keeper/end_block_server_game.go#L34-L41]
        storedGame, found = k.GetStoredGame(ctx, storedGameId)
        if !found {
            panic("Fifo head game not found " + nextGame.FifoHead)
        }
        deadline, err := storedGame.GetDeadlineAsTime()
        if err != nil {
            panic(err)
        }
        ```
    * Test for expiry:
        ```go [https://github.com/cosmos/b9-checkers-academy-draft/blob/7739537804f350241f59ee55b443a66b68883fc8/x/checkers/keeper/end_block_server_game.go#L42]
        if deadline.Before(ctx.BlockTime()) {
            // TODO
        } else {
            // All other games come after anyway
            break
        }
        ```
        * If it has expired, remove it from the FIFO:
            ```go [https://github.com/cosmos/b9-checkers-academy-draft/blob/7739537804f350241f59ee55b443a66b68883fc8/x/checkers/keeper/end_block_server_game.go#L44]
            k.RemoveFromFifo(ctx, &storedGame, &nextGame)
            ```
        * Then check whether the game is worth keeping at all. If yes, then set the winner as the opponent of the player whose turn it is and save:
            ```go [https://github.com/cosmos/b9-checkers-academy-draft/blob/7739537804f350241f59ee55b443a66b68883fc8/x/checkers/keeper/end_block_server_game.go#L45-L55]
            if storedGame.MoveCount == 0 {
                storedGame.Winner = rules.NO_PLAYER.Color
                // No point in keeping a game that was never played
                k.RemoveStoredGame(ctx, storedGameId)
            } else {
                storedGame.Winner, found = opponents[storedGame.Turn]
                if !found {
                    panic(fmt.Sprintf(types.ErrCannotFindWinnerByColor.Error(), storedGame.Turn))
                }
                k.SetStoredGame(ctx, storedGame)
            }
            ```
        * Emit the relevant event:
            ```go [https://github.com/cosmos/b9-checkers-academy-draft/blob/7739537804f350241f59ee55b443a66b68883fc8/x/checkers/keeper/end_block_server_game.go#L56-L63]
            ctx.EventManager().EmitEvent(
                sdk.NewEvent(sdk.EventTypeMessage,
                    sdk.NewAttribute(sdk.AttributeKeyModule, types.ModuleName),
                    sdk.NewAttribute(sdk.AttributeKeyAction, types.ForfeitGameEventKey),
                    sdk.NewAttribute(types.ForfeitGameEventIdValue, storedGameId),
                    sdk.NewAttribute(types.ForfeitGameEventWinner, storedGame.Winner),
                ),
            )
            ```
        * Move along the FIFO for the next run of the loop:
            ```go [https://github.com/cosmos/b9-checkers-academy-draft/blob/7739537804f350241f59ee55b443a66b68883fc8/x/checkers/keeper/end_block_server_game.go#L65]
            storedGameId = nextGame.FifoHead
            ```
* After the loop has ended, do not forget to save the latest FIFO state:
    ```go [https://github.com/cosmos/b9-checkers-academy-draft/blob/7739537804f350241f59ee55b443a66b68883fc8/x/checkers/keeper/end_block_server_game.go#L72]
    k.SetNextGame(ctx, nextGame)
    ```

And that's all there is to it. For an explanation as to why this setup is resistant to attack from an unbounded number of expired games, see [A Game FIFO](./03-starport-09-game-fifo.md).