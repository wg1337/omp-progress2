/*
	progress2.inc - player progress bars for open.mp

	Originally written by Fl\Uffffffffo Toribio, maintained by Southclaws (v2+).
	This version is standalone: it needs only <open.mp>, no YSI and no other includes.

	Changes from Southclaws/progress2:
	  - y_iterate removed: bar slots are tracked with the per-bar `is_created` flag.
	  - y_hooks removed: OnPlayerDisconnect and the script exit callback use plain ALS hooks,
	    which chain correctly with y_hooks or any other hook method used elsewhere.
	  - Bars are reset on disconnect even without YSI (previously leaked to the next player).
	  - playerid is bounds-checked in every public function.
	  - Updates move textdraws with PlayerTextDrawSetPos instead of destroying/recreating them,
	    so textdraw IDs never change and value updates only touch the value textdraw.
	  - Filler alpha uses a clamp instead of a bitwise AND.
	  - min == max no longer divides by zero.
	  - Textdraw creation failures are detected and rolled back.
	  - SetPlayerProgressBarValue skips the update when the value did not change.
	  - New SetPlayerProgressBarRange(playerid, barid, min, max) sets both with one redraw.

	Options (define before including):
	  MAX_PLAYER_BARS        maximum bars per player, e.g. #define MAX_PLAYER_BARS (PlayerBar:30)
	  PROGRESS2_NO_LOG       silence error messages printed to the server console
*/

#if defined _progress2_included
	#endinput
#endif
#define _progress2_included

#include <open.mp>

#if defined MAX_PLAYER_BARS
	#if (MAX_PLAYER_BARS - PlayerBar:0) == PlayerBar:0
		#error You forgot to set value of MAX_PLAYER_BARS. Type something like: #define MAX_PLAYER_BARS (PlayerBar:30).
	#endif
	#if (MAX_PLAYER_BARS < PlayerBar:1)
		#error You are trying to allocate invalid number of PlayerBars. Set value of MAX_PLAYER_BARS greater than 0.
	#elseif MAX_PLAYER_BARS > PlayerBar:(_:MAX_PLAYER_TEXT_DRAWS / 3)
		#error You are trying to allocate too many PlayerBars. Set value of MAX_PLAYER_BARS below or equal to (MAX_PLAYER_TEXT_DRAWS / 3).
	#endif
#else
	#define MAX_PLAYER_BARS (PlayerBar:(_:MAX_PLAYER_TEXT_DRAWS / 3))
#endif

// NOTE: this is a NaN, so `value == INVALID_PLAYER_BAR_VALUE` is never true.
// Check IsValidPlayerProgressBar() before calling getters instead.
#define INVALID_PLAYER_BAR_VALUE (Float:0xFFFFFFFF)
#define INVALID_PLAYER_BAR_ID    (PlayerBar:-1)

#if defined PROGRESS2_NO_LOG
	#define _p2_err(%0); {}
#else
	#define _p2_err(%0); printf("[progress2] " %0);
#endif

enum progressbar_direction {
	BAR_DIRECTION_RIGHT,
	BAR_DIRECTION_LEFT,
	BAR_DIRECTION_HORIZONTAL_FROM_0,
	BAR_DIRECTION_UP,
	BAR_DIRECTION_DOWN,
	BAR_DIRECTION_VERTICAL_FROM_0,
};

// Upper bound of the filler (inner background) alpha.
#define _P2_FILL_MAX_ALPHA (0x66)
// Horizontal correction applied to inner boxes, in canvas pixels.
#define _P2_CORRECTION_X   (1.25)

enum E_BAR_DATA {
	bool:is_created,
	bool:pbar_show,
	Float:pbar_posX,
	Float:pbar_posY,
	Float:pbar_width,
	Float:pbar_height,
	pbar_colour,
	Float:pbar_minValue,
	Float:pbar_maxValue,
	Float:pbar_progressValue,
	Float:E_PADDING_X,
	Float:E_PADDING_Y,
	progressbar_direction:pbar_direction
}

enum E_BAR_TEXT_DRAW {
	PlayerText:pbar_back,
	PlayerText:pbar_fill,
	PlayerText:pbar_main
}

enum E_PROGRESSBAR_BOUNDRY {
	Float:E_PBAR_BACKGROUND_POS_X,
	Float:E_PBAR_BACKGROUND_POS_Y,
	Float:E_PBAR_BACKGROUND_RIGHT,
	Float:E_PBAR_BACKGROUND_HEIGHT,
	Float:E_PBAR_FILLER_POS_X,
	Float:E_PBAR_FILLER_POS_Y,
	Float:E_PBAR_FILLER_RIGHT,
	Float:E_PBAR_FILLER_HEIGHT,
	Float:E_PBAR_VALUE_POS_X,
	Float:E_PBAR_VALUE_POS_Y,
	Float:E_PBAR_VALUE_RIGHT,
	Float:E_PBAR_VALUE_HEIGHT,
}

static PlayerText:pbar_TextDraw[MAX_PLAYERS][MAX_PLAYER_BARS][E_BAR_TEXT_DRAW];
new pbar_Data[MAX_PLAYERS][MAX_PLAYER_BARS][E_BAR_DATA];

/*
	Internal
*/

// LEFT, UP and VERTICAL_FROM_0 grow towards negative screen coordinates.
static stock bool:_p2_isReversed(progressbar_direction:direction) {
	return (direction == BAR_DIRECTION_LEFT || direction == BAR_DIRECTION_UP || direction == BAR_DIRECTION_VERTICAL_FROM_0);
}

static stock bool:_p2_isFromZero(progressbar_direction:direction) {
	return (direction == BAR_DIRECTION_HORIZONTAL_FROM_0 || direction == BAR_DIRECTION_VERTICAL_FROM_0);
}

// True if the value box has a visible size for the current value.
static stock bool:_p2_needDrawValue(playerid, PlayerBar:barid) {
	new
		Float:cur_value = pbar_Data[playerid][barid][pbar_progressValue],
		Float:min_value = pbar_Data[playerid][barid][pbar_minValue],
		Float:max_value = pbar_Data[playerid][barid][pbar_maxValue];

	if (max_value <= min_value) {
		return false;
	}
	if (_p2_isFromZero(pbar_Data[playerid][barid][pbar_direction])) {
		if (max_value < 0.0) {
			return (cur_value < max_value);
		}
		if (min_value > 0.0) {
			return (cur_value > min_value);
		}
		return (floatabs(cur_value) > 0.001 * (max_value - min_value));
	}
	return (cur_value > min_value);
}

static stock _p2_getRatios(progressbar_direction:direction, Float:cur_value, Float:min_value, Float:max_value, &Float:ratio_from, &Float:ratio_to) {
	new Float:range_value = max_value - min_value;
	if (range_value <= 0.0) {
		// Empty or inverted range: draw nothing instead of dividing by zero.
		ratio_from = 0.0;
		ratio_to = 0.0;
		return;
	}
	if (_p2_isFromZero(direction)) {
		if (max_value < 0.0) {
			ratio_from = 1.0;
		} else if (min_value > 0.0) {
			ratio_from = 0.0;
		} else {
			ratio_from = -min_value / range_value;
		}
	} else {
		ratio_from = 0.0;
	}
	ratio_to = (cur_value - min_value) / range_value;
}

static stock _p2_sortPair(&Float:a, &Float:b) {
	if (a > b) {
		new Float:tmp = b;
		b = a;
		a = tmp;
	}
}

static stock _p2_computeBoundary(playerid, PlayerBar:barid, boundary[E_PROGRESSBAR_BOUNDRY]) {
	new
		Float:pos_x = pbar_Data[playerid][barid][pbar_posX],
		Float:pos_y = pbar_Data[playerid][barid][pbar_posY],
		Float:padding_x = pbar_Data[playerid][barid][E_PADDING_X],
		Float:padding_y = pbar_Data[playerid][barid][E_PADDING_Y],
		Float:width = pbar_Data[playerid][barid][pbar_width],
		Float:height = pbar_Data[playerid][barid][pbar_height],
		progressbar_direction:direction = pbar_Data[playerid][barid][pbar_direction],
		bool:is_vertical = (direction > BAR_DIRECTION_HORIZONTAL_FROM_0),
		Float:size_multiplier = _p2_isReversed(direction) ? -1.0 : 1.0,
		Float:outer_pos_x2 = pos_x + width,
		Float:outer_pos_y2 = pos_y + height,
		Float:inner_pos_x1 = pos_x + padding_x,
		Float:inner_pos_x2 = outer_pos_x2 - padding_x,
		Float:inner_pos_y1 = pos_y + padding_y,
		Float:inner_pos_y2 = outer_pos_y2 - padding_y,
		Float:inner_size_x = inner_pos_x2 - inner_pos_x1,
		Float:inner_size_y = inner_pos_y2 - inner_pos_y1,
		Float:value_pos_x1, Float:value_pos_y1,
		Float:value_pos_x2, Float:value_pos_y2,
		Float:ratio_from, Float:ratio_to;

	_p2_getRatios(direction,
		pbar_Data[playerid][barid][pbar_progressValue],
		pbar_Data[playerid][barid][pbar_minValue],
		pbar_Data[playerid][barid][pbar_maxValue],
		ratio_from, ratio_to);
	ratio_from *= size_multiplier;
	ratio_to *= size_multiplier;

	if (is_vertical) {
		value_pos_x1 = inner_pos_x1;
		value_pos_x2 = inner_pos_x2;
	} else {
		value_pos_y1 = inner_pos_y1;
		value_pos_y2 = inner_pos_y2;
	}

	switch (direction) {
		case BAR_DIRECTION_RIGHT:             { value_pos_x1 = inner_pos_x1; }
		case BAR_DIRECTION_LEFT:              { value_pos_x1 = inner_pos_x2; }
		case BAR_DIRECTION_HORIZONTAL_FROM_0: { value_pos_x1 = inner_pos_x1; }
		case BAR_DIRECTION_UP:                { value_pos_y1 = inner_pos_y2; }
		case BAR_DIRECTION_DOWN:              { value_pos_y1 = inner_pos_y1; }
		case BAR_DIRECTION_VERTICAL_FROM_0:   { value_pos_y1 = inner_pos_y2; }
	}

	if (is_vertical) {
		value_pos_y2 = value_pos_y1;
		value_pos_y1 += inner_size_y * ratio_from;
		value_pos_y2 += inner_size_y * ratio_to;
	} else {
		value_pos_x2 = value_pos_x1;
		value_pos_x1 += inner_size_x * ratio_from;
		value_pos_x2 += inner_size_x * ratio_to;
	}

	_p2_sortPair(value_pos_x1, value_pos_x2);
	_p2_sortPair(value_pos_y1, value_pos_y2);

	value_pos_x1 += _P2_CORRECTION_X;
	value_pos_x2 -= _P2_CORRECTION_X;
	inner_pos_x1 += _P2_CORRECTION_X;
	inner_pos_x2 -= _P2_CORRECTION_X;

	boundary[E_PBAR_BACKGROUND_POS_X] = pos_x;
	boundary[E_PBAR_BACKGROUND_POS_Y] = pos_y;
	boundary[E_PBAR_BACKGROUND_RIGHT] = outer_pos_x2;
	boundary[E_PBAR_BACKGROUND_HEIGHT] = 0.1 * (outer_pos_y2 - pos_y);
	boundary[E_PBAR_FILLER_POS_X] = inner_pos_x1;
	boundary[E_PBAR_FILLER_POS_Y] = inner_pos_y1;
	boundary[E_PBAR_FILLER_RIGHT] = inner_pos_x2;
	boundary[E_PBAR_FILLER_HEIGHT] = 0.1 * (inner_pos_y2 - inner_pos_y1);
	boundary[E_PBAR_VALUE_POS_X] = value_pos_x1;
	boundary[E_PBAR_VALUE_POS_Y] = value_pos_y1;
	boundary[E_PBAR_VALUE_RIGHT] = value_pos_x2;
	boundary[E_PBAR_VALUE_HEIGHT] = 0.1 * (value_pos_y2 - value_pos_y1);
}

static stock PlayerBar:_p2_findFree(playerid) {
	for (new PlayerBar:barid = PlayerBar:0; barid < MAX_PLAYER_BARS; ++barid) {
		if (!pbar_Data[playerid][barid][is_created]) {
			return barid;
		}
	}
	return INVALID_PLAYER_BAR_ID;
}

// Forget every bar of a player without touching textdraws (used on disconnect,
// where the server has already destroyed the player's textdraws).
static stock _p2_resetPlayer(playerid) {
	for (new PlayerBar:barid = PlayerBar:0; barid < MAX_PLAYER_BARS; ++barid) {
		pbar_Data[playerid][barid][is_created] = false;
		pbar_Data[playerid][barid][pbar_show] = false;
		pbar_TextDraw[playerid][barid][pbar_back] = INVALID_PLAYER_TEXT_DRAW;
		pbar_TextDraw[playerid][barid][pbar_fill] = INVALID_PLAYER_TEXT_DRAW;
		pbar_TextDraw[playerid][barid][pbar_main] = INVALID_PLAYER_TEXT_DRAW;
	}
}

static stock _p2_destroyTextDraws(playerid, PlayerBar:barid) {
	for (new E_BAR_TEXT_DRAW:slot = E_BAR_TEXT_DRAW:0; slot < E_BAR_TEXT_DRAW; ++slot) {
		if (pbar_TextDraw[playerid][barid][slot] != INVALID_PLAYER_TEXT_DRAW) {
			PlayerTextDrawDestroy(playerid, pbar_TextDraw[playerid][barid][slot]);
			pbar_TextDraw[playerid][barid][slot] = INVALID_PLAYER_TEXT_DRAW;
		}
	}
}

// Creates the three textdraws once. Back is created first so it is drawn below
// fill, and fill below main. Later updates only move them, so the order holds.
static stock bool:_p2_createTextDraws(playerid, PlayerBar:barid) {
	new boundary[E_PROGRESSBAR_BOUNDRY];
	_p2_computeBoundary(playerid, barid, boundary);

	pbar_TextDraw[playerid][barid][pbar_back] = CreatePlayerTextDraw(playerid,
		boundary[E_PBAR_BACKGROUND_POS_X], boundary[E_PBAR_BACKGROUND_POS_Y], "_");
	pbar_TextDraw[playerid][barid][pbar_fill] = CreatePlayerTextDraw(playerid,
		boundary[E_PBAR_FILLER_POS_X], boundary[E_PBAR_FILLER_POS_Y], "_");
	pbar_TextDraw[playerid][barid][pbar_main] = CreatePlayerTextDraw(playerid,
		boundary[E_PBAR_VALUE_POS_X], boundary[E_PBAR_VALUE_POS_Y], "_");

	if (
		pbar_TextDraw[playerid][barid][pbar_back] == INVALID_PLAYER_TEXT_DRAW ||
		pbar_TextDraw[playerid][barid][pbar_fill] == INVALID_PLAYER_TEXT_DRAW ||
		pbar_TextDraw[playerid][barid][pbar_main] == INVALID_PLAYER_TEXT_DRAW
	) {
		_p2_destroyTextDraws(playerid, barid);
		return false;
	}

	_p2_applyGeometry(playerid, barid, boundary, true);
	_p2_applyColour(playerid, barid);
	return true;
}

// Recomputes geometry and pushes it to the client. `all` = also move the
// background and filler (needed when position, size or padding changed).
static stock _p2_update(playerid, PlayerBar:barid, bool:all) {
	new boundary[E_PROGRESSBAR_BOUNDRY];
	_p2_computeBoundary(playerid, barid, boundary);
	_p2_applyGeometry(playerid, barid, boundary, all);

	if (pbar_Data[playerid][barid][pbar_show]) {
		if (all) {
			PlayerTextDrawShow(playerid, pbar_TextDraw[playerid][barid][pbar_back]);
			PlayerTextDrawShow(playerid, pbar_TextDraw[playerid][barid][pbar_fill]);
		}
		PlayerTextDrawShow(playerid, pbar_TextDraw[playerid][barid][pbar_main]);
	}
}

static stock _p2_applyGeometry(playerid, PlayerBar:barid, const boundary[E_PROGRESSBAR_BOUNDRY], bool:all) {
	new PlayerText:td;
	if (all) {
		td = pbar_TextDraw[playerid][barid][pbar_back];
		PlayerTextDrawSetPos(playerid, td, boundary[E_PBAR_BACKGROUND_POS_X], boundary[E_PBAR_BACKGROUND_POS_Y]);
		PlayerTextDrawTextSize(playerid, td, boundary[E_PBAR_BACKGROUND_RIGHT], 0.0);
		PlayerTextDrawLetterSize(playerid, td, 1.0, boundary[E_PBAR_BACKGROUND_HEIGHT]);
		PlayerTextDrawUseBox(playerid, td, true);

		td = pbar_TextDraw[playerid][barid][pbar_fill];
		PlayerTextDrawSetPos(playerid, td, boundary[E_PBAR_FILLER_POS_X], boundary[E_PBAR_FILLER_POS_Y]);
		PlayerTextDrawTextSize(playerid, td, boundary[E_PBAR_FILLER_RIGHT], 0.0);
		PlayerTextDrawLetterSize(playerid, td, 1.0, boundary[E_PBAR_FILLER_HEIGHT]);
		PlayerTextDrawUseBox(playerid, td, true);
	}

	td = pbar_TextDraw[playerid][barid][pbar_main];
	PlayerTextDrawSetPos(playerid, td, boundary[E_PBAR_VALUE_POS_X], boundary[E_PBAR_VALUE_POS_Y]);
	PlayerTextDrawTextSize(playerid, td, boundary[E_PBAR_VALUE_RIGHT], 0.0);
	PlayerTextDrawLetterSize(playerid, td, 1.0, boundary[E_PBAR_VALUE_HEIGHT]);
	PlayerTextDrawUseBox(playerid, td, _p2_needDrawValue(playerid, barid));
}

static stock _p2_applyColour(playerid, PlayerBar:barid) {
	new
		colour = pbar_Data[playerid][barid][pbar_colour],
		alpha = colour & 0xFF;

	// Background: black, with the bar's alpha.
	PlayerTextDrawBoxColour(playerid, pbar_TextDraw[playerid][barid][pbar_back], alpha);
	// Filler: the bar's colour at half its alpha, capped at _P2_FILL_MAX_ALPHA.
	PlayerTextDrawBoxColour(playerid, pbar_TextDraw[playerid][barid][pbar_fill], (colour & 0xFFFFFF00) | min(alpha / 2, _P2_FILL_MAX_ALPHA));
	// Value: the bar's colour.
	PlayerTextDrawBoxColour(playerid, pbar_TextDraw[playerid][barid][pbar_main], colour);
}

forward PlayerBar:CreatePlayerProgressBar(
	playerid,
	Float:x, Float:y,
	Float:width = 55.5, Float:height = 3.2,
	colour = 0xFF1C1CFF,
	Float:max = 100.0,
	progressbar_direction:direction = BAR_DIRECTION_RIGHT
);
forward Float:GetPlayerProgressBarValue(playerid, PlayerBar:barid);

/*
	Public API
*/

stock bool:IsValidPlayerProgressBar(playerid, PlayerBar:barid) {
	if (playerid < 0 || playerid >= MAX_PLAYERS) {
		return false;
	}
	if (barid < PlayerBar:0 || barid >= MAX_PLAYER_BARS) {
		return false;
	}
	return pbar_Data[playerid][barid][is_created];
}

stock PlayerBar:CreatePlayerProgressBar(
	playerid,
	Float:x, Float:y,
	Float:width = 55.5, Float:height = 3.2,
	colour = 0xFF1C1CFF,
	Float:max = 100.0,
	progressbar_direction:direction = BAR_DIRECTION_RIGHT
) {
	if (!IsPlayerConnected(playerid)) {
		_p2_err("attempt to create progress bar for invalid player %d", playerid);
		return INVALID_PLAYER_BAR_ID;
	}
	if (direction < BAR_DIRECTION_RIGHT || direction > BAR_DIRECTION_VERTICAL_FROM_0) {
		_p2_err("invalid progress bar direction %d", _:direction);
		return INVALID_PLAYER_BAR_ID;
	}

	new PlayerBar:barid = _p2_findFree(playerid);
	if (barid == INVALID_PLAYER_BAR_ID) {
		_p2_err("MAX_PLAYER_BARS limit reached for player %d", playerid);
		return INVALID_PLAYER_BAR_ID;
	}

	pbar_Data[playerid][barid][pbar_show] = false;
	pbar_Data[playerid][barid][pbar_posX] = x;
	pbar_Data[playerid][barid][pbar_posY] = y;
	pbar_Data[playerid][barid][pbar_width] = width;
	pbar_Data[playerid][barid][pbar_height] = height;
	pbar_Data[playerid][barid][pbar_colour] = colour;
	pbar_Data[playerid][barid][pbar_minValue] = 0.0;
	pbar_Data[playerid][barid][pbar_maxValue] = max;
	pbar_Data[playerid][barid][pbar_progressValue] = 0.0;
	pbar_Data[playerid][barid][pbar_direction] = direction;
	pbar_Data[playerid][barid][E_PADDING_X] = 1.2;
	pbar_Data[playerid][barid][E_PADDING_Y] = 1.0;

	if (!_p2_createTextDraws(playerid, barid)) {
		_p2_err("could not create textdraws for player %d (textdraw limit reached?)", playerid);
		return INVALID_PLAYER_BAR_ID;
	}

	pbar_Data[playerid][barid][is_created] = true;
	return barid;
}

stock bool:DestroyPlayerProgressBar(playerid, PlayerBar:barid) {
	if (!IsValidPlayerProgressBar(playerid, barid)) {
		return false;
	}

	_p2_destroyTextDraws(playerid, barid);
	pbar_Data[playerid][barid][is_created] = false;
	pbar_Data[playerid][barid][pbar_show] = false;
	return true;
}

stock DestroyAllPlayerProgressBars(playerid) {
	for (new PlayerBar:barid = PlayerBar:0; barid < MAX_PLAYER_BARS; ++barid) {
		DestroyPlayerProgressBar(playerid, barid);
	}
	return 1;
}

stock bool:IsPlayerProgressBarVisible(playerid, PlayerBar:barid) {
	return (IsValidPlayerProgressBar(playerid, barid) && pbar_Data[playerid][barid][pbar_show]);
}

stock bool:ShowPlayerProgressBar(playerid, PlayerBar:barid) {
	if (!IsValidPlayerProgressBar(playerid, barid)) {
		return false;
	}

	pbar_Data[playerid][barid][pbar_show] = true;

	PlayerTextDrawShow(playerid, pbar_TextDraw[playerid][barid][pbar_back]);
	PlayerTextDrawShow(playerid, pbar_TextDraw[playerid][barid][pbar_fill]);
	PlayerTextDrawShow(playerid, pbar_TextDraw[playerid][barid][pbar_main]);
	return true;
}

stock bool:HidePlayerProgressBar(playerid, PlayerBar:barid) {
	if (!IsValidPlayerProgressBar(playerid, barid)) {
		return false;
	}

	pbar_Data[playerid][barid][pbar_show] = false;

	PlayerTextDrawHide(playerid, pbar_TextDraw[playerid][barid][pbar_back]);
	PlayerTextDrawHide(playerid, pbar_TextDraw[playerid][barid][pbar_fill]);
	PlayerTextDrawHide(playerid, pbar_TextDraw[playerid][barid][pbar_main]);
	return true;
}

// Position

stock bool:GetPlayerProgressBarPos(playerid, PlayerBar:barid, &Float:x, &Float:y) {
	if (!IsValidPlayerProgressBar(playerid, barid)) {
		return false;
	}
	x = pbar_Data[playerid][barid][pbar_posX];
	y = pbar_Data[playerid][barid][pbar_posY];
	return true;
}

stock bool:SetPlayerProgressBarPos(playerid, PlayerBar:barid, Float:x, Float:y) {
	if (!IsValidPlayerProgressBar(playerid, barid)) {
		return false;
	}
	pbar_Data[playerid][barid][pbar_posX] = x;
	pbar_Data[playerid][barid][pbar_posY] = y;
	_p2_update(playerid, barid, true);
	return true;
}

// Width

stock Float:GetPlayerProgressBarWidth(playerid, PlayerBar:barid) {
	if (!IsValidPlayerProgressBar(playerid, barid)) {
		return INVALID_PLAYER_BAR_VALUE;
	}
	return pbar_Data[playerid][barid][pbar_width];
}

stock bool:SetPlayerProgressBarWidth(playerid, PlayerBar:barid, Float:width) {
	if (!IsValidPlayerProgressBar(playerid, barid)) {
		return false;
	}
	pbar_Data[playerid][barid][pbar_width] = width;
	_p2_update(playerid, barid, true);
	return true;
}

// Height

stock Float:GetPlayerProgressBarHeight(playerid, PlayerBar:barid) {
	if (!IsValidPlayerProgressBar(playerid, barid)) {
		return INVALID_PLAYER_BAR_VALUE;
	}
	return pbar_Data[playerid][barid][pbar_height];
}

stock bool:SetPlayerProgressBarHeight(playerid, PlayerBar:barid, Float:height) {
	if (!IsValidPlayerProgressBar(playerid, barid)) {
		return false;
	}
	pbar_Data[playerid][barid][pbar_height] = height;
	_p2_update(playerid, barid, true);
	return true;
}

// Colour

stock GetPlayerProgressBarColour(playerid, PlayerBar:barid) {
	if (!IsValidPlayerProgressBar(playerid, barid)) {
		return 0;
	}
	return pbar_Data[playerid][barid][pbar_colour];
}

stock bool:SetPlayerProgressBarColour(playerid, PlayerBar:barid, colour) {
	if (!IsValidPlayerProgressBar(playerid, barid)) {
		return false;
	}
	pbar_Data[playerid][barid][pbar_colour] = colour;
	_p2_applyColour(playerid, barid);

	if (pbar_Data[playerid][barid][pbar_show]) {
		ShowPlayerProgressBar(playerid, barid);
	}
	return true;
}

// Min / max / value

stock Float:GetPlayerProgressBarMinValue(playerid, PlayerBar:barid) {
	if (!IsValidPlayerProgressBar(playerid, barid)) {
		return INVALID_PLAYER_BAR_VALUE;
	}
	return pbar_Data[playerid][barid][pbar_minValue];
}

// Clamps `value` to the bar's range, stores it and redraws the value textdraw.
// Without `force`, nothing is sent when the stored value would not change.
static stock _p2_setValue(playerid, PlayerBar:barid, Float:value, bool:force) {
	new
		Float:min_value = pbar_Data[playerid][barid][pbar_minValue],
		Float:max_value = pbar_Data[playerid][barid][pbar_maxValue];

	if (value < min_value) {
		value = min_value;
	} else if (value > max_value) {
		value = max_value;
	}
	if (!force && value == pbar_Data[playerid][barid][pbar_progressValue]) {
		return;
	}
	pbar_Data[playerid][barid][pbar_progressValue] = value;

	// Only the value textdraw changes, so only it is moved and re-shown.
	_p2_update(playerid, barid, false);
}

stock bool:SetPlayerProgressBarMinValue(playerid, PlayerBar:barid, Float:min_value) {
	if (!IsValidPlayerProgressBar(playerid, barid)) {
		return false;
	}
	pbar_Data[playerid][barid][pbar_minValue] = min_value;
	_p2_setValue(playerid, barid, pbar_Data[playerid][barid][pbar_progressValue], true);
	return true;
}

stock Float:GetPlayerProgressBarMaxValue(playerid, PlayerBar:barid) {
	if (!IsValidPlayerProgressBar(playerid, barid)) {
		return INVALID_PLAYER_BAR_VALUE;
	}
	return pbar_Data[playerid][barid][pbar_maxValue];
}

stock bool:SetPlayerProgressBarMaxValue(playerid, PlayerBar:barid, Float:max) {
	if (!IsValidPlayerProgressBar(playerid, barid)) {
		return false;
	}
	pbar_Data[playerid][barid][pbar_maxValue] = max;
	_p2_setValue(playerid, barid, pbar_Data[playerid][barid][pbar_progressValue], true);
	return true;
}

// Sets min and max together with a single redraw. Returns false if min > max.
stock bool:SetPlayerProgressBarRange(playerid, PlayerBar:barid, Float:min_value, Float:max_value) {
	if (!IsValidPlayerProgressBar(playerid, barid) || min_value > max_value) {
		return false;
	}
	pbar_Data[playerid][barid][pbar_minValue] = min_value;
	pbar_Data[playerid][barid][pbar_maxValue] = max_value;
	_p2_setValue(playerid, barid, pbar_Data[playerid][barid][pbar_progressValue], true);
	return true;
}

// Does nothing (and sends nothing) if the clamped value equals the current one,
// so calling it every tick is cheap.
stock bool:SetPlayerProgressBarValue(playerid, PlayerBar:barid, Float:value) {
	if (!IsValidPlayerProgressBar(playerid, barid)) {
		return false;
	}
	_p2_setValue(playerid, barid, value, false);
	return true;
}

stock Float:GetPlayerProgressBarValue(playerid, PlayerBar:barid) {
	if (!IsValidPlayerProgressBar(playerid, barid)) {
		return INVALID_PLAYER_BAR_VALUE;
	}
	return pbar_Data[playerid][barid][pbar_progressValue];
}

// Direction

stock progressbar_direction:GetPlayerProgressBarDirection(playerid, PlayerBar:barid) {
	if (!IsValidPlayerProgressBar(playerid, barid)) {
		return BAR_DIRECTION_RIGHT;
	}
	return pbar_Data[playerid][barid][pbar_direction];
}

stock bool:SetPlayerProgressBarDirection(playerid, PlayerBar:barid, progressbar_direction:direction) {
	if (!IsValidPlayerProgressBar(playerid, barid)) {
		return false;
	}
	if (direction < BAR_DIRECTION_RIGHT || direction > BAR_DIRECTION_VERTICAL_FROM_0) {
		return false;
	}
	pbar_Data[playerid][barid][pbar_direction] = direction;
	_p2_update(playerid, barid, false);
	return true;
}

// Padding

stock bool:GetPlayerProgressBarPadding(playerid, PlayerBar:barid, &Float:padding_x, &Float:padding_y) {
	if (!IsValidPlayerProgressBar(playerid, barid)) {
		return false;
	}
	padding_x = pbar_Data[playerid][barid][E_PADDING_X];
	padding_y = pbar_Data[playerid][barid][E_PADDING_Y];
	return true;
}

stock bool:SetPlayerProgressBarPadding(playerid, PlayerBar:barid, Float:padding_x, Float:padding_y) {
	if (!IsValidPlayerProgressBar(playerid, barid)) {
		return false;
	}
	pbar_Data[playerid][barid][E_PADDING_X] = padding_x;
	pbar_Data[playerid][barid][E_PADDING_Y] = padding_y;
	_p2_update(playerid, barid, true);
	return true;
}

// Kept for compatibility with the original API.
stock PlayerBar:PlayerBarUI_FindFree(playerid) {
	if (playerid < 0 || playerid >= MAX_PLAYERS) {
		return INVALID_PLAYER_BAR_ID;
	}
	return _p2_findFree(playerid);
}

// Kept for compatibility: forgets a player's bars without destroying textdraws.
stock PlayerBarUI_ResetPlayer(playerid) {
	if (0 <= playerid < MAX_PLAYERS) {
		_p2_resetPlayer(playerid);
	}
}

stock PlayerBarUI_ResetAll() {
	for (new playerid = 0; playerid < MAX_PLAYERS; ++playerid) {
		_p2_resetPlayer(playerid);
	}
}

/*
	Hooks (plain ALS, no YSI)
*/

public OnPlayerDisconnect(playerid, reason) {
	// The server destroys player textdraws on disconnect; just forget the bars
	// so the next player with this ID starts clean.
	_p2_resetPlayer(playerid);

	#if defined P2_OnPlayerDisconnect
		return P2_OnPlayerDisconnect(playerid, reason);
	#else
		return 1;
	#endif
}
#if defined _ALS_OnPlayerDisconnect
	#undef OnPlayerDisconnect
#else
	#define _ALS_OnPlayerDisconnect
#endif
#define OnPlayerDisconnect P2_OnPlayerDisconnect
#if defined P2_OnPlayerDisconnect
	forward P2_OnPlayerDisconnect(playerid, reason);
#endif

#if defined FILTERSCRIPT
	// Player textdraws outlive a filterscript unload, so clean them up.
	public OnFilterScriptExit() {
		for (new playerid = 0; playerid < MAX_PLAYERS; ++playerid) {
			if (IsPlayerConnected(playerid)) {
				DestroyAllPlayerProgressBars(playerid);
			}
		}
		#if defined P2_OnFilterScriptExit
			return P2_OnFilterScriptExit();
		#else
			return 1;
		#endif
	}
	#if defined _ALS_OnFilterScriptExit
		#undef OnFilterScriptExit
	#else
		#define _ALS_OnFilterScriptExit
	#endif
	#define OnFilterScriptExit P2_OnFilterScriptExit
	#if defined P2_OnFilterScriptExit
		forward P2_OnFilterScriptExit();
	#endif
#else
	public OnGameModeExit() {
		for (new playerid = 0; playerid < MAX_PLAYERS; ++playerid) {
			if (IsPlayerConnected(playerid)) {
				DestroyAllPlayerProgressBars(playerid);
			}
		}
		#if defined P2_OnGameModeExit
			return P2_OnGameModeExit();
		#else
			return 1;
		#endif
	}
	#if defined _ALS_OnGameModeExit
		#undef OnGameModeExit
	#else
		#define _ALS_OnGameModeExit
	#endif
	#define OnGameModeExit P2_OnGameModeExit
	#if defined P2_OnGameModeExit
		forward P2_OnGameModeExit();
	#endif
#endif
