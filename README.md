# Connect Four Script Reference

## board.gd

```gdscript
extends Node2D
# allows access to position and drawing fucntions, etc

#   CONSTANTS
# these values will never change 
# giving them variable names will help keep the code readable

# row 0 is the top, row 5 is the bottom
const rows = 6
# col 0 is the left, col 6 is the right
const cols = 7

const player = 1
const ai = 2
const empty = 0   # a cell containing 0 means nobody has played there
const cell_size = 80  # each cell is 80x80 pixels on screen
const ai_depth = 5 # how many moves ahead the AI looks (higher = smarter but slower)

#   SIGNALS
#   UImain can listen to these and react
#   without board.gd needing to know anything about them
#   this keeps the board script focused on game logic only

signal turn_changed(who) # fired every time the active player switches
signal game_over(winner) # fired when someone wins or the board is full

#   VARIABLES

var grid: Array = []           # a 2D array of 0s, 1s, and 2s
                               # this is the real game state. the visuals just mirror it
var current_player = player  # whose turn it is currently
var game_end = false         # flips to true when the game is over

# SETUP

func _ready():  
    _init_grid()  # build the logical 2D array first
    _draw_board()  # then draw the visual board on screen

# builds the 2D array that tracks the game state
func _init_grid():  
    grid = []       # resets the board every time
    for row in range(rows):   # loop 0 to 5
        var new_row = []      # create a fresh inner array for this row
        for col in range(cols):   # loop 0 to 6
            new_row.append(empty) # fill every cell with 0 (empty)
        grid.append(new_row)   # add the completed row to the grid
        
func _draw_board(): # creates the visual representation of the board
                    # for each of the 42 cells it creates a ColorRect node 
                    # and adds it as a child of the board node
    for row in range(rows): 
        for col in range(cols):
            var cell = ColorRect.new()  #intstantiates a colorRect node to be stored for later
            cell.size = Vector2(cell_size -4, cell_size -4) 
            # make the cell 76x76 instead of 80x80
            # the 4px gap (2px on each side) shows the board background
            # color through the gaps, creating a visible grid effect
            cell.position = Vector2(col * cell_size + 2, row * cell_size +2)
            # place the cell at the correct position relative to the board node
            # col * cell_size gives the left edge of this column in pixels
            # + 2 shifts it right by 2px to center the gap
            cell.color = Color(0.15, 0.35, 0.75) #uses r,g,b , is meant to be overwritten 
                                                    #when placing a piece
            cell.name = "Cell_%d_%d" % [row, col]  # "%d" is a format placeholder for an integer
                                                   # gives this node a unique name 
                                    #is meant for not having to hold all 42 spots of data
            add_child(cell)
            # adds the colorrect into the scene tree as a child of board
            
            
    #   drops a piece into the given column for the current player
    #   simulates gravity by scanning from the bottom row upward
    #   and placing the piece in the first empty cell found
    #   updates both the logical grid and the visual display
    #   returns true if the piece was placed, false if column full

func place_piece(col: int) -> bool:
    for row in range(rows -1, -1, -1):  # range counts down
        # start at the bottom row (5) and work upward (toward 0)
        # because gravity pulls pieces to the lowest available row
        if grid[row][col] == empty:  # check if this cell is unoccupied
            grid[row][col] = current_player  # write the current player's value into the grid
            _cell_upd_vis(row, col, current_player) # update the visual
            return true
    return false
    
    
    # scans the entire board to see if the given player has
    # four pieces in a row in the four possible directions
    
    # if a window starts too close to an edge it would go
    # out of bounds when adding 1, 2, or 3 to the index
    # each direction restricts its starting range so the
    # full window of 4 always stays within the valid grid
    
func check_win(who: int) -> bool:
    # check horizontal      # start up to column 3. if col started at 4, col+3
                            # would be 7 which is out of bounds (max is 6).
                            # range(cols - 3) = range(4) = [0, 1, 2, 3]
    for row in range(rows):
        for col in range(cols - 3): # the \ continues the line
            if grid[row][col] == who and \
            grid[row][col+1] == who and \
            grid[row][col+2] == who and \
            grid[row][col+3] == who:
                return true

    # check vertical # row can only start up to row 2
                     # if row started at 3, row+3 would be 6 which is out of bounds (max is 5)
                     # range(rows - 3) = range(3) = [0, 1, 2]
    for row in range(rows - 3):
        for col in range(cols):
            if grid[row][col] == who and \
               grid[row+1][col] == who and \
               grid[row+2][col] == who and \
               grid[row+3][col] == who:
                return true
                
        # check down right   # both row and col increase together 
                             # row max = 2, col max = 3.
    for row in range(rows - 3):
        for col in range(cols - 3):
            if grid[row][col] == who and \
               grid[row+1][col+1] == who and \
               grid[row+2][col+2] == who and \
               grid[row+3][col+3] == who:
                return true

    # check down left
                # row increases (bounded to max 2) but col decreases
                # because col decreases by up to 3, col must start
                # at minimum 3 so that col-3 never goes below 0
                # range(3, cols) = [3, 4, 5, 6]
    for row in range(rows - 3):
        for col in range(3, cols):
            if grid[row][col] == who and \
               grid[row+1][col-1] == who and \
               grid[row+2][col-2] == who and \
               grid[row+3][col-3] == who:
                return true

    return false
    
func _cell_upd_vis(row: int, col: int, who: int):  #color code swaps the color of the pieces
                                                    #depending on whose turn it is
    var cell = get_node("Cell_%d_%d" % [row, col])
    if who == player:
        cell.color = Color(0.95, 0.2, 0.2)    # red for player
    elif who == ai:
        cell.color = Color(0.95, 0.85, 0.1)   # yellow for AI

func switch_turns(): #  flips current_player between player and ai,
                     #  then emits turn_changed so UImain updates the label.
                     #  The emit must come after the flip so the signal carries
                     #  the new player value, not the old one
    if current_player == player:
        current_player = ai
    else:
        current_player = player
    emit_signal("turn_changed", current_player)
        
            #  Godot calls this automatically every single time any
            #  input event occurs (clicking, key presses, mouse movement)
func _input(event):
    if game_end:
        return  # exit immediately if the game is finished
    if event is InputEventMouseButton and event.pressed:
        if event.button_index == MOUSE_BUTTON_LEFT and current_player == player:
            # check current_player == player so the ai's
            # automated turns cannot accidentally trigger it
            var col = int((event.position.x - position.x) / cell_size)
            # subtract position.x first so that x=0 aligns with the
            # left edge of the board and not the left edge of the window
            # dividing by cell_size converts pixels to a list of the columns
            if col >= 0 and col < cols:
                # make sure the click landed inside the board
                _handle_move(col)
                # handles what happens after the player clicks a column

func _handle_move(col: int):
    if place_piece(col):
            # place_piece returns false if the column is full, then the click is ignored
        if check_win(player):
            # check if the piece just placed wins the game
            game_end = true 
            emit_signal("game_over", player)  
            # tell the UI to show "You win!" and the restart button
            return
        switch_turns()
            # ai will take its turn here
        await get_tree().create_timer(0.3).timeout 
            # small delay for 0.3 seconds before the ai move
            # get_tree() accesses the scene tree
            # gives the player a moment to see their piece land
        _ai_turn()
        #switches turn to ai
        
        if _is_board_full():
            game_end = true
            emit_signal("game_over", 0)
                                        # 0 means draw 
            return

# AI turn - minimax and best move 

func _ai_turn():
    var best_col = _get_best_move()  # _get_best_move() runs minimax on every valid column
                                     # and returns the index of the column with the highest score
    place_piece(best_col)
                                # place the ai's piece in the chosen column
    if check_win(ai):
        game_end = true 
        emit_signal("game_over", ai) 
        # tell the UI to show "You lose!" and the restart button
        return
    switch_turns()

#  loops through every valid column, temporarily places an
#  ai piece, runs minimax to score the resulting position,
#  then undoes the placement. returns the column with the
#  highest score with the move minimax recommends.

func _get_best_move() -> int:
    var best_score = -INF   # INF is a built in GDScript constant for infinity
                            # -INF is negative infinity aka the lowest possible score
                            # starts here so any real score will be higher
    var best_col = 0  # default fallback column
    for col in range(cols):
        if _is_valid_move(col):   # skips full columns
            var row = _get_next_open_row(col) # find the row the piece would land on in this column
            grid[row][col] = ai
            # temporarily write an ai piece into the grid
            # this modifies the real grid, but undoes it right after
            # minimax needs the grid to reflect the hypothetical move
            var score = _minimax(grid, ai_depth - 1, -INF, INF, false)
            # score this position with minimax.
            # ai_depth - 1 because this placement counts as depth 1
            # false means the next turn is the player's (minimizing)
            grid[row][col] = empty  # undo the temporary placement, restoring the grid
            if score > best_score:
                best_score = score  # if this column scored better than everything so far,
                                    # remember it as the new best choice
                best_col = col  
    return best_col


#   the ai algorithm -

# simulates both players playing optimally many moves ahead
# and returns a score representing how good the current
# board state is for the ai

# works by building a game tree
# each node is a board state, each branch is a possible move
# the AI explores branches to a given depth
# takes the branching path that will ensure the lowest 
# score for player and highest score for ai

# alpha-beta pruning speeds this up by abandoning branches
# that can never affect the final decision

func _minimax(board: Array, depth: int, alpha: float, beta: float, maximizing: bool) -> float:
    # base cases - stop solving
    if check_win(ai):   return 100000 + depth   
    # the ai has already won in this simulated state
    # add depth because a win found at higher remaining depth
    # means it happens sooner in the real game
    if check_win(player): return -100000 - depth 
    # the player has won in this simulated state
    # subtract depth to penalise losses that happen sooner
    if depth == 0 or _is_board_full():
        return _score_board()   
        # either hit search limit or the board is completely full
        # instead of recursing further, evaluate the current position
        # with the heuristic function _score_board() 
        #(instead of exhaustively exploring all possibilities,
        # heuristics guide the search by narrowing down the most promising paths)

    if maximizing:   # the ai is choosing a move
        var max_eval = -INF  # start with the worst possible value for the maximizer
        for col in range(cols):
            if _is_valid_move(col):
                var row = _get_next_open_row(col)
                board[row][col] = ai     # temporarily place an ai piece to simulate this move
                var eval = _minimax(board, depth - 1, alpha, beta, false)
                # now it's the player's turn (false = minimizing)
                # depth decreases by 1 each level until a base case is hit
                board[row][col] = empty
                # undo the simulated move before trying the next column
                max_eval = max(max_eval, eval)
                # keep the highest score seen so far across all columns
                alpha = max(alpha, eval)
                # alpha = best score the ai has found on any branch so far
                if beta <= alpha:
                    break   # pruning: the player already has a move elsewhere that
                # results in a score <= alpha for the ai. the player will
                # never choose this branch because the ai does better here
                # so we can stop evaluating remaining columns, they won't
                # change the outcome the player would choose
        return max_eval
    else:                    # player's turn
        var min_eval = INF  # start with the worst possible value for the minimizer
        for col in range(cols):
            if _is_valid_move(col):
                var row = _get_next_open_row(col)
                board[row][col] = player # temporarily place a player piece to simulate this move
                var eval = _minimax(board, depth - 1, alpha, beta, true)
                # recurse, now it is the ai's turn (true = maximizing)
                board[row][col] = empty
                # undo the simulated move
                min_eval = min(min_eval, eval)
                # keep the lowest score seen so far. the player wants
                # to minimize the ai's score
                beta = min(beta, eval)
                # beta = best score the player has found on any branch so far
                if beta <= alpha:
                # pruning: the ai already has a move elsewhere that results
                # in a score >= beta. the ai will never choose this branch
                # stop evaluating 
                    break   
        return min_eval

func _score_board() -> float:
    # score center column more (better strategic position)
    var score: float = 0.0
    var center_col: int = cols / 2
    for row in range(rows):
        if grid[row][center_col] == ai:
            score += 3
            # add 3 points for each ai piece in the center column
            # this nudges the ai toward center control 
    return score

func _is_valid_move(col: int) -> bool:
    return grid[0][col] == empty
    # a column is full when its top row (row 0) is occupied
    # if the top cell is empty, at least one space remains below

func _get_next_open_row(col: int) -> int:
    for row in range(rows - 1, -1, -1):
        # scan from the bottom row (5) upward to row 0
        # the first empty cell from the bottom is where a dropped
        # piece would land
        if grid[row][col] == empty:
            return row  # return immediately when we find the lowest empty row
    return -1
    # the column is completely full

func _is_board_full() -> bool:
    for col in range(cols):
        if _is_valid_move(col):
            return false
            # found at least one column that still has space
            # the board is not full, it returns false immediately
    return true
        # every column was checked and none had a valid move, 
        # game is a draw since every column is full


extends Node2D

# this script controls the UI 
# listens to signals from board.gd and reacts visually


# @onready means it will wait until the scene is fully loaded, then it uses the nodes
# the $ is shorthand for get_node() - it finds a node by its path in the scene tree


@onready var turn_label = $UI/TurnLabel  # UI text 
@onready var restart_button = $UI/RestartButton  # restart button when game ends
@onready var board = $Board  #listens for the board node to then use its signals

func _ready():   # runs once automatically when the scene loads
    restart_button.pressed.connect(_on_restart)
     # when the restart button is clicked, run _on_restart
    board.game_over.connect(_on_game_over)
     # when board.gd emits game_over, run _on_game_over
    board.turn_changed.connect(_on_turn_changed)
     # when board.gd emits turn_changed, run _on_turn_changed

# who = integer board.gd sent with the signal (1 = player, 2 = ai)
func _on_turn_changed(who: int):
    if who == 1:
        turn_label.text = "Your turn"
        turn_label.modulate = Color(0.95, 0.2, 0.2)  # color for player
    else:
        turn_label.text = "AI thinking..."
        turn_label.modulate = Color(0.95, 0.85, 0.1)  # color for ai

# winner = integer board.gd sent with the signal (1 = player, 2 = ai, 0 = draw)
func _on_game_over(winner: int):
    if winner == 1:
        turn_label.text = "You win!"
    elif winner == 2:
        turn_label.text = "You lose!"
    else:
        turn_label.text = "It's a draw!"
    restart_button.visible = true

func _on_restart():
    get_tree().reload_current_scene()
      # get_tree() accesses godot's scene tree
      # reloads the scene to start new
