# ENGLISH VERSION OF README FILE
# The Dry Peaks API
**Christos Sapounas**

## Technologies

* **Backend:** PHP
* **Database:** MySQL / MariaDB
* **Data Exchange:** JSON

---

## Overview

"The Dry Peaks API" is a project for the course *Web Systems and Applications Development*. The API is built around the card game **Xeri(Ξερή)**, a classic Greek card game.

### Game Rules

1. Each player is dealt **6 cards** and **4 cards** are placed face-up on the table.
2. The table cards are visible, but players can only play on top of the **first/top card**.
3. On their turn, a player throws a card onto the table and **collects all table cards** if:
   - The thrown card matches the top card on the table.
   - The thrown card is a **JACK** (value = 12).
   - **XERI**: If there is only **one card** on the table and the player throws a matching card — that's a Xeri!
4. When players run out of cards, each is dealt **4 new cards**.
5. The game ends when there are no more cards on the table, in the deck, or in the players' hands.

The API implements all game rules and exposes the core functions needed by a frontend to render the GUI — including `create`, `reset`, `end`, and `update` game operations, player moves (throwing a card), user management, and various getters for game state.

Communication between frontend and backend is done via **JSON**. For example, to call `getgame()`, the frontend sends:

```json
{
  "gameid": "0123456789"
}
```

And the backend responds with:

```json
{
  "gameid": "0123456789",
  "p1": "p123123",
  "p2": "p223123",
  "turn": "p123123",
  "deckofcards": "{json with deck of cards}",
  "boardofcards": "{json with board cards}"
}
```

---

## Files & How They Work

### `TheDryPeaks.php`
The **entry point** of the API. All frontend requests go through this file. It contains a `switch` statement that routes requests to the appropriate handler based on the URL segments between `/`.

For example, to call `getgame()`, the frontend makes a `GET` request to `/TheDryPeaks.php/game/game` with the game ID in the request body. The router then executes:

```php
if(isset($input['gameid'])){
    $data = getgame($input['gameid']);
    echo json_encode($data);
} else {
    http_response_code(400);
    echo json_encode(['error' => 'Missing game id to get game']);
}
```

This pattern is repeated for all API functions.

---

### `user.php`
Contains all functions related to user management.

| Function | Description |
|----------|-------------|
| `createuser($username, $pword)` | Inserts a new user into the database with a unique ID. Returns `INSERT_SUCCESS` and the new user ID. |
| `deleteuser($userid)` | Deletes a user. Returns `DELETE_SUCCESS` and the deleted user's ID. |
| `checkCredentials($username, $password)` | Validates user login credentials. |
| `login($userid)` | Issues an authentication token to the user after credentials are verified. |
| `logout($userid)` | Revokes the user's token (logs them out). |
| `getIn_game_state($userid)` | Returns whether the user is currently in a game. |
| `getusername($userid)` | Returns the user's username. |
| `getpoints($userid)` | Returns the user's total points. |
| `getUsersGameID($userid)` | Returns the game ID of the game the user is currently in. |
| `getuserStats($userid)` | Returns the user's username, state, and points. |

---

### `conn.php`
Connects the PHP application to the MySQL database. Requires a `dbpassag.php` file containing the database credentials:

```php
<?php
  $DB_USER = '';
  $DB_PASS = '';
  $DB_SOCKET = '';
  $DB = '';
  $DB_PORT = ;
?>
```

---

### `match.php`
Handles matchmaking — placing users into a queue to find an opponent. The main function to use is `inqueue($userid)`, which handles everything automatically:

- If another player is already waiting in the queue → returns `GAME_JOINED` and the existing game ID.
- If no player is waiting → creates a new game and returns `GAME_CREATED` and the new game ID.

The deck is stored in the database when the first player creates the game, and is used when the second player joins.

---

### `game.php`
The core game file. Contains all functions for creating, resetting, fetching, and updating games, as well as the main gameplay logic.

| Function | Description |
|----------|-------------|
| `creategame($user1, $user2)` | Creates a new game by calling `sharecards()`, storing all data in the DB, and updating both players' `in_game_state`. Returns the game ID and the second player's cards. |
| `endgame($gameid)` | Deletes the game from the database. Returns `DELETE_SUCCESS`. |
| `resetgame($gameid, $usr1, $usr2)` | Restarts the game with the same players but a new game ID. |
| `updategame($gameid, $cards, $turn, $boardcards, $gamepoints, $userid, $cardsgathered)` | Updates the game state in the database with new data. |
| `updategamepoints($userid, $gamepoints)` | Updates the player's points. |
| `finishgame($gameid)` | Called before `endgame()`. Calculates the winner (or draw) and saves results to `log_game_file` — a history of all completed games. |
| `sharecards()` | Loads cards from `deck.json`, shuffles them, and distributes them to Player 1, Player 2, the board, and the remaining deck. |
| `throwcard($gameid, $cardnumber, $token)` | The most important game function. Validates the token, identifies whose turn it is, applies the appropriate game rule based on the top board card, checks if players need new cards or if the game is over, and if the game ends — calls `finishgame()`. Returns the action taken (e.g. `GAINED_CARDS`, `NO_ACTION`), the status (e.g. `PLAYING`, `GAME_OVER`, `ROUND_ENDED`), the card played, and the card it was played on top of. |
| `RoundShareCards($gameid)` | Called at the end of each round to deal new cards from the stored deck to both players. |
| `changetoken($gameid)` | Changes the token of the player whose turn it is. Should be called after every `throwcard()` for security. |

---

### `status.php`
Contains a single function that returns the current game status of a user, identified only by their token.

---

## Recommended API Usage Flow

> (Assumes both users are already logged in with valid tokens)

**1. Player 1 joins the queue:**

`POST /TheDryPeaks.php/match/queue`

Response:
```json
{
  "status": "GAME_CREATED",
  "gameid": "697bf75650193"
}
```

**2. Player 2 joins the queue and gets matched:**

`POST /TheDryPeaks.php/match/queue`

Response:
```json
{
  "status": "GAME_JOINED",
  "gameid": "697bf75650193"
}
```

**3. A player throws a card:**

`POST /TheDryPeaks.php/game/throwcard`

Request:
```json
{
  "gameid": "697bf75650193",
  "number": 0,
  "token": "tokenexample"
}
```

Response:
```json
{
  "status": "PLAYING",
  "Card_we_played": { "suit": "hearts", "rank": "9", "value": 9 },
  "On_top_of": { "suit": "spades", "rank": "2", "value": 2 },
  "action": "NO_ACTION"
}
```

**4. After each move, call `changetoken` to rotate the active player's token for security.**

**5. When the game ends, `throwcard` returns `GAME_OVER`:**

```json
{
  "status": "GAME_OVER",
  "Card_we_played": { "suit": "diamonds", "rank": "King", "value": 13 },
  "On_top_of": { "suit": "hearts", "rank": "3", "value": 3 },
  "action": "GAINED_CARDS"
}
```

**6. Fetch game results:**

`GET /TheDryPeaks.php/game/results`

Response:
```json
{
  "log_id": "131231321",
  "time_stamp": "2026-01-30 15:17:16",
  "user_1_id": "634636",
  "user_1_score": 32,
  "user_2_id": "7869867966",
  "user_2_score": 56,
  "winner": "7869867966"
}
```

**7. Delete the game:**

`DELETE /TheDryPeaks.php/game/endgame`

Response:
```json
{
  "status": "DELETE_SUCCESS"
}
```

# GREEK VERSION OF README FILE
# The Dry Peaks API (Ξερή)
**Χρήστος Σαπουνάς**

## Τεχνολογίες

* **Backend:** PHP 
* **Database:** MySQL / MariaDB
* **Data Exchange:** JSON

### Περίληψη API
Το "The Dry Peaks API" αποτελεί ένα project για το μάθημα Ανάπτυξη Διαδικτυακών  
Συστημάτων και Εφαρμογών. Το API αφορά το παιχνιδι της ξερής το οποίο είναι ένα κλασσικό παιχνίδι τράπουλας.
Βασικοί κανόνες της ξερής είναι οι εξής: 
1. Μοιράζονται 6 χαρτία στους παίχτες και 4 στο "τραπέζι"
2. Τα χαρτιά στο τραπέζι είναι φανερά αλλα παίζουμε μόνο πάνω στο πρώτο
3. Ο παίχτης ο οποίος έχει σειρά ρίχνει ένα χαρτί στο τραπέζι και μαζεύει τα φυλλα αν:
  * Το χαρτί του παίχτη είναι ίδιο με αυτό στο τραπέζι
  * Το χαρτί του παίχτι είναι JACK(value=12)
  * ΞΕΡΗ: Αν υπάρχει μόνο ένα φύλλο κάτω και ρίξει τοτε ο παίχτης το ιδιο τότε έχουμε ξερή!
5. Οταν τελιώσουν τα χαρτιά των παιχτών τους μοιράζουμε απο 4 κάρτες στον καθένα
4. Παίζουμε μεχρι να μην υπάρχουν αλλα χαρτιά στο τραπέζι, στην τράπουλα και στους παίχτες

Στο API υλοποιούνται οι κανόνες, καθώς και οι βασικές λειτουργίες που θα μπορέσει να χρησιμοποιήσει το frontend
ώστε να υλοποιήθει και το GUI. Βασικές λειτουργίες όπως create,reset,end και update game. Μαζί με τις βασικές λειτουργίες θα υπάρχουν και οι κινήσεις μπορεί να κάνει ο παίχτης(στην προκειμένη περίπτωση να ρίξει μία κάρτα στο τραπέζι). Επιπλέον, διαθέτει συναρτήσεις που μπορεί να χρησιμοποιήσει το frontend για την επεξεργασία ενός χρήστη. Επισής έχουν ενσωματωθεί οι κατάλληλοι getters που θα μπορέσει να χρησιμοποιήσει το frontend όπου χρειαστεί, για παράδειγμα αν χρειάζεται τα στοιχεία του τρέχων παιχνιδιού τότε θα καλέσει απο τον server την συνάρτηση getgame($gameid) ή το status του παιχνιδιού που βρίσκεται ένας παίχτης με το αρχείο status.php. Η τεχνολογία με την οποία επικοινωνεί το frontend και το backend είναι με json, δηλαδη στο παραπάνω παράδειγμα μας για να καλέσει την συναρτηση το frontend θα πρέπει να δωσει ένα json με το gameid: 
```json 
{
  "gameid":"0123456789"
}
```
και το backend θα επιστέψει πάλι json με τα στοιχεία τα οποία ζητήσαμε:
```json 
{
  "gameid":"0123456789",
  "p1" :"p123123",
  "p2" :"p223123",
  "turn" :"p123123",
  "deckofcards" :"{json with deck of cards}",
  "boardofcards" :"{jason with deck of cards}"
}
```

### Αρχεία & Τρόπος λειτουργείας Αρχείων

**TheDryPeaks.php:** Το αρχείο TheDryPeaks.php είναι το entry point του API. Είναι το αρχείο το οποίο καλείται απο το frontend αν θέλει να καλέσει μια συνάρτηση απο τον server. Υπάρχει ένα switch το οποίο ανακατευθύνει κατάλληλο κομματι του κώδικα. Για να κληθεί η συνάρτηση getgame() που προαναφέραμε, θα πρέπει να κληθεί το /TheDryPeaks.php/game/game με μέθοδο GET και με τα request ανάμεσα απο το τις '/' ανακατευθυνόμαστε στο κομμάτι κώδικα:
```php
  if(isset($input['gameid'])){
    $data = getgame($input['gameid']);
    echo json_encode($data);
  }else {
    http_response_code(400);
    echo json_encode(['error' => 'Missing game id to get game']);
  }
```
Όπου καλούμε την συνάρτηση απο το game.php, το ίδιο επαναλαμβάνεται για όλες τις συναρτήσεις του API.

**user.php:** Στο αρχείο user.php βρίσκονται οι συναρτήσεις που αφορούν τους χρήστες που έχουμε αποθηκευμένους στην βάση μας.
  * **createuser($username,$pword):** Συνάρτηση που εισχωρούμε τον χρήστη στην βάση και του αναθέτουμε ένα μοναδικό ID μαζί με το username και το password και επιστρέφει status επιτυχίας, "INSERT_SUCCESS" μαζι με το μοναδικό ID του χρήστη.
  * **deleteuser($userid):** Συνάρτηση διαγραφής χρήστη που ως status επιτυχίας, "DELETE_SUCCESS" μαζί με το id του χρήστη που διαγράψαμε.
  * **checkCredentials($username, $password):** Συνάρτηση που μπορούμε να καλέσουμε για να ελέγξουμε αν τα στοιχεία που έβαλε ο χρήστης είναι σωστά ώστε να μπορέσουμε να τον συνέδουμε στον λογαριασμό του.
  * **login($userid):** Αφού έχει γίνει το check για τα στοιχεία του χρήστη, θα πρέπει να γίνει ένα login ώστε να πάρει token ο χρήστης.
  * **logout($userid):** Όταν θα θέλει να αποσυνδεθεί ο χρήστης τότε θα διαγράφεται το token.
  * **getIn_game_state($userid):** Επιστρέφει αν ο χρήστης παίζει παιχνίδι εκείνη την στιγμή.
  * **getusername($userid):** Επιστρέφει τον username του χρήστη.
  * **getpoints($userid):** Επιστρέφει τα points του χρήστη.
  * **getUsersGameID($userid):** Επιστρέφει το gameid του παιχνιδιου που βρίσκεται ο χρήστης.
  * **getuserStats($userid):** Επιστρέφει το username,state και τους πόντους του χρήστη.

**conn.php:** Συνδέει το php αρχείο μας με την mysql βάση και για να επιτύχουμε σύνδεση χρείαζόμαστε ένα αρχείο 'dbpassag.php' το οποίο περιέχει τα credentials.
```php
<?php 
  $DB_USER ='';
  $DB_PASS ='';
  $DB_SOCKET ='';
  $DB='';
  $DB_PORT=;
?>
```
**match.php:** Βρίσκονται οι συναρτήσεις που θα χρειαστεί το front end για να εισχωρίσει τον χρήστη σε ουρα παιχνιδιου εύρεσης αντιπάλου. Το μόνο που χρειάζεται να χρησιμοποιηθεί απο το αρχείο αυτό είναι η συνάρτηση inqueue($userid) που "κανονίζει" τα πάντα, δηλαδή μόλις καλεστεί η συνάρτηση τότε ψάχνουμε να βρούμε άλλους παίχτες στην ουρά. Αν υπάρχουν τότε η συναρτηση μας γυρίζει status "GAME_JOINED" και το game id που μόλις συνδέθηκε, αλλιώς δημιουργούμε παιχνίδι και επιστρέφουμε status "GAME_CREATED" και το game id που μόλις δημιουργήθηκε. Εφόσον το παιχνίδι δημιουργείται πριν βρεθεί δεύτερος παιχτης, οι κάρτες του αποθηκεύνται στην βάση μας και θα τις χρησιμοποιούμε όταν συνδέεται και ο 2ος παίχτης.

**game.php:** Στο αρχείο game βρίσκονται οι συναρτήσεις για την δημιουργία,επανεκκίνηση,λήψη και ενημέρωση παιχνιδιου καθώς και οι κύριες συναρτήσεις του παιχνιδίου που προαναφέραμε καθώς και ορισμένοι getters.
  * **creategame($user1,$user2):** Η συνάρτηση όπου δημιουργούμε το παιχνίδι καλώντας πρώτα την συνάρτηση sharecards() η οποία μας επιστρέφει τις κάρτες παίχτη1,παιχτη2, τις καρτες που θα μοιραστούν στο τραπέζι, και τις κάρτες της τράπουλας. Τέλος εισχωρούμε τα δεδομένα στη βάση δημιουργώντας ένα νέο παιχνίδι αναθέτοντας τις κάρτες στον παίχτη και αλλάζοντας το in_game_state του παίχτη. Επιστρέφουμε game id και τις κάρτες του δεύτερου παίχτη.
  * **endgame($gameid):** Διαγράφουμε το παιχνίδι απο την βάση και επιστρέφουμε status "DELETE_SUCCESS".
  * **resetgame($gameid, $usr1, $usr2):** Επανεκινούμε το παιχνίδι με τους ίδιους παίχτες αλλά με διαφορετικό game id.
  * **updategame($gameid, $cards, $turn, $boardcards, $gamepoints,$userid,$cardsgathered):** Ενημερώνουμε το game στην βάση με καινουργια δεδομένα.
  * **updategamepoints($userid,$gamepoints):** Ανανεώνουμε τους πόντους του παίχτη.
  * **finishgame($gameid):** Η συνάρτηση finishgame καλείται πριν την συνάρτηση endgame καθώς στην συνάρτηση υπολογίζουμε τον νικητή ή την ισσοπαλία του παιχνιδιού και τα αποθηκεύουμε στην βάση μας σε ένα log_game_file στο οποίο βρίσκονται όλα τα παιχνίδια που έχουν παίξει στην εφαρμογή μας στον παρελθον.
  * **sharecards():** Η συνάρτηση η οποία καλείται για να μοιραστούν οι κάρτες στην αρχή του παιχνιδιού. Η συνάρτηση παιρνει τις κάρτες οι οποίες είναι στο αρχείο deck.json, τις "ανακατεύει" και τις επιστρέφει αναλόγως το μερίδιο που δικαιούται ο παίχτης 1, παίχτης 2 κ.λ.π.
  * **throwcard($gameid,$cardnumber, $token):** Η πιο σημαντική συνάρτηση του παιχνιδιού. Πρώτα γίνεται έλεγχος αν το token valid.Στην συνέχεια του throwcard το frontend έχει στείλει το gameid και το νούμερο της κάρτας (μαζί με το token) που θέλει να ρίξει ο χρήστης και μέσω της βάσης γνωρίζουμε ποιός έχει σειρά να παίξει.
  Ανάλογα με την κάρτα που βρίσκεται στην κορυφη των καρτών στο τραπέζι εφαρμόζεται και ο κατάλληλος κανόνας που αναφέρθηκε στην περίληψη. Πριν απο κάθε επιστροφή της συνάρτησης ελέγχουμε αν οι παίχτες έχουν κάρτες για να τους ξαναμοιράσουμε κάρτες ή αν το παιχνίδι ΤΕΛΕΊΩΣΕ. Σε αυτό το σημείο αν το παιχνίδι έχει τελειώσει τότε υπολογίζονται όπως ποιός θα πάρει τις καρτες οι οποίες είναι κάτω και ποιός θα πάρει τους +3 πόντους και μετά απο όλα αυτά καλείται η συνάρτηση finishgame() ώστε να υπολογιστεί ο νικητής(ή η ισοπαλία) και να δημιουργηθεί το log για το παιχνίδι που μόλις τελείωσε.Η συνάρτηση επιστρέφει την ενέργεια που έγινε(π.χ."GAINED_CARDS","NO_ACTION"κ.λ.π.), το status(π.χ."PLAYING","GAME_OVER","ROUND_ENDED"), την κάρτα που έπαιξε ο παίχτης και την κάρτα που έπαιξε "πάνω".
  * **RoundShareCards($gameid):** Η συνάρτηση που καλούμε μετά το πέρας του κάθε γύρου η οποία παίρνει κάρτες απο την τράπουλα που είναι αποθηκευμένη στην βάση και μοιράζει κάρτες στους παίχτες.
  * **changetoken($gameid):** Πολύ σημαντική συνάρτηση για την ασφάλεια γιατι αλλάζει το token του χρήστη που έχει σειρά. Η ιδανική χρήση είναι μετά το throwcard() ώστε να ανατεθεί νέο token στον παίχτη που έχει σειρά.

**status.php:** To οποίο περιέχει μόνο μια συναρτηση η οποία στέλνει στο status του χρήστη και έχοντας μόνο το token

## Προτεινόμενος τρόπος χρήσης API

* (Έστω οτι ο κάθε χρήστης είναι συνδεμένος στον λογαριασμό του)
* Αρχικά ο πρώτος χρήστης επιλέγει(/TheDryPeaks.php/match/queue ) να μπει σε ουρά για παιχνιδι και επιστρέφεται απο την συνάρτηση:
```json 
{
  "status": "GAME_CREATED",
  "gameid": "697bf75650193"
}
```
* Επιλέγει και ένας δεύτερος χρήστης να μπεί στην ουρά για παιχνίδι και εφόσον βρεθεί ένα παιχνίδι που έχει χώρο τότε επιστρέφεται απο την συνάρτηση:
```json 
{
  "status": "GAME_JOINED",
  "gameid": "697bf75650193"
}
```
* Αν ο παίχτης θέλει να παίξει την πρώτη κάρτα τότε θα πρέπει να γίνει POST(/TheDryPeaks.php/game/throwcard ) μαζί με το token του παίχτη, το gameid και το αριθμό της κάρτας που θέλει να παίξει:
```json 
{
  "gameid":"697bf75650193",
  "number":0,
  "token":"tokenexample"
}
```
και η συνάρτηση θα επιστρέψει:
```json 
{
  "status": "PLAYING",
  "Card_we_played": {
    "suit": "hearts",
    "rank": "9",
    "value": 9
  },
  "On_top_of": {
    "suit": "spades",
    "rank": "2",
    "value": 2
  },
  "action": "NO_ACTION"
}
```
* Μετά απο κάθε κίνηση για λόγους ασφάλειας θα είναι καλύτερο να καλεστεί η συνάρτηση change token ώστε να αλλάξει το token του παίχτη που επρόκειται να παίξει.
* Όταν θα τελειώσει το παιχνίδι η συνάρτηση throwcard() επιστρέφει status "GAME_OVER":
```json 
{
  "status": "GAME_OVER",
  "Card_we_played": {
    "suit": "diamonds",
    "rank": "King",
    "value": 13
  },
  "On_top_of": {
    "suit": "hearts",
    "rank": "3",
    "value": 3
  },
  "action": "GAINED_CARDS"
}
```
* Υπάρχει δυνατότητα να εμφανιστούν τα αποτελέσματα του παιχνιδιού με την κλήση GET (/TheDryPeaks.php/game/results) όπου επιστρέφεται το log:
```json 
{
  "log_id": "131231321",
  "time_stamp": "2026-01-30 15:17:16",
  "user_1_id": "634636",
  "user_1_score": 32,
  "user_2_id": "7869867966",
  "user_2_score": 56,
  "winner": "7869867966"
}
```
* Τέλος θα χρειαστεί να διαγραφεί το παιχνίδι:
```json 
{
  "status": "DELETE_SUCCESS"
}
```

  






