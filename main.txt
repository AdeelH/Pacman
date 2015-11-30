#include <stdio.h>
#include <stdlib.h>
#include <time.h>
#include <curses.h>

typedef unsigned char byte;

const int HEIGHT, WIDTH;

const byte img[] = {2, ' ', ' ', '.', 186, 205, 201, 187, 200, 188, 203, 202,
185, 204, 206};

// index in img[]
#define PAC           0
#define BUFFER        1
#define PATH_EMPTY    2
#define PATH_DOT      3
#define WALL_VER      4
#define WALL_HOR      5
#define WALL_COR_UL   6
#define WALL_COR_UR   7
#define WALL_COR_LL   8
#define WALL_COR_LR   9
#define WALL_T       10
#define WALL_TI      11
#define WALL_TL      12
#define WALL_TR      13
#define WALL_X       14

static byte* map;

typedef enum {NONE, LEFT, UP, RIGHT, DOWN} direction;

typedef struct
{
	byte x, y;
	byte col;
	byte trail;
	direction opp_last_move;
}
Game_Char;


#define EN_MAX 3    // max # of enemies allowed (overrides # in map file)

const byte en_cols[] = {COLOR_RED, COLOR_GREEN, COLOR_MAGENTA};
Game_Char* pac;
Game_Char* enemies[EN_MAX];

int dot_count, score;
byte game_active;

// func declarations
void get_map(char* map_file);
void draw_walls();
void fill_map();
void play();
void pac_move(byte x1, byte y1);
void en_move();
direction en_move_decide(Game_Char* en);
void en_move_dir(Game_Char* en, direction d);
void draw_ch(Game_Char* ch);

// inlined functions
void en_move_vec(Game_Char* en, byte x, byte y);
direction opp_dir(direction dir);
byte not_backtracking(Game_Char* en, direction d);
byte legal_move(byte x, byte y);
byte legal_dir(Game_Char* ch, direction d);
void draw_trail(Game_Char* ch);


int main()
{
    initscr();
    raw();
    noecho();
    curs_set(0);
    keypad(stdscr, TRUE);

	start_color();
	init_pair(1, COLOR_CYAN, COLOR_BLACK);
	init_pair(2, COLOR_YELLOW, COLOR_BLACK);

    srand(time(NULL));
    do
    {
        // let user choose map
        clear();
        mvprintw(0, 0, "Press 0, 1, or 2 to select map.");
        char fname[20];
        sprintf(fname, "maps/%c.txt", getch());
        clear();
        // set up game
        get_map(fname);
        resize_term(HEIGHT, WIDTH+10);
        draw_walls();
        fill_map();

        // play
        play();

        // cleanup
        free(map);
        free(pac); pac = NULL;
        int i;
        for (i=0; i < EN_MAX; ++i)
            free(enemies[i]);

        // play again?
        resize_term(HEIGHT+2, WIDTH+10);
        mvprintw(HEIGHT, 0, "Press 'r' to play again.");
    }
    while(getch() == 'r');
    endwin();

    return 0;
}

void get_map(char* map_file)
{
    FILE* mapf = fopen(map_file, "r");
    if (!mapf)
    {
        fclose (mapf);
        printw("Error opening file.");
        getch();
		exit(-1);
    }
    fscanf(mapf, "%d", &HEIGHT);
    fscanf(mapf, "%d", &WIDTH);

    map = (byte*)malloc(sizeof(byte)*HEIGHT*WIDTH);
    char c;
    int i = 0;
	dot_count = 0;
    byte en_count = 0;
    while (i < HEIGHT*WIDTH && !ferror(mapf) && (c = fgetc(mapf)) != EOF)
    {
		switch(c)
		{
            case 'd': map[i++] = PATH_DOT; ++dot_count; break;
			case 'b': map[i++] = BUFFER; break;
			case 'v': map[i++] = WALL_VER; break;
			case 'h': map[i++] = WALL_HOR; break;
			case 'r': map[i++] = WALL_COR_UL; break;
			case '7': map[i++] = WALL_COR_UR; break;
			case 'l': map[i++] = WALL_COR_LL; break;
			case 'j': map[i++] = WALL_COR_LR; break;
			case 't': map[i++] = WALL_T; break;
			case 'i': map[i++] = WALL_TI; break;
			case '3': map[i++] = WALL_TL; break;
			case 'k': map[i++] = WALL_TR; break;
			case 'p': if (pac != NULL)
                      {
                        printw("Error: Too many pacmen.");
                        getch(); exit(-3);
                      }
                      pac = (Game_Char*)malloc(sizeof(Game_Char));
                      if (!pac) exit(-2);
                      pac->y = i/WIDTH;
                      pac->x = i%WIDTH;
                      pac->col = 2;
                      pac->trail = PATH_EMPTY;
                      map[i++] = PATH_EMPTY;
                      break;
			case 'e': if (en_count < EN_MAX)
                      {
                          enemies[en_count] = (Game_Char*)malloc(sizeof(Game_Char));
                          enemies[en_count]->opp_last_move = NONE;
                          enemies[en_count]->trail = PATH_DOT;
                          // set color
                          byte tmp = en_count%EN_MAX;
                          enemies[en_count]->col = tmp + 3;
                          init_pair(tmp + 3, en_cols[tmp], COLOR_BLACK);
                          // set position
                          enemies[en_count]->y = i/WIDTH;
                          enemies[en_count++]->x = i%WIDTH;
                      }
                      map[i++] = PATH_DOT;
                      ++dot_count;
                      break;
		}
    }
    fclose (mapf);

	if (dot_count <= 0)
	{
		printw("Error: no dots found in map %s", map_file);
		getch();
		exit(-3);
	}
}

void draw_walls()
{
	attron(COLOR_PAIR(1));

    int i, j, x = 0;
    for (i = 0; i < HEIGHT; ++i)
    {
        for (j = 0; j < WIDTH; ++j, ++x)
			if (map[x] >= WALL_VER)
				mvaddch(i, j,  img[map[x]] | A_ALTCHARSET);
    }
	attroff(COLOR_PAIR(1));
}

void fill_map()
{
    attron(COLOR_PAIR(2));
    int i, j, x = 0;
    for (i = 0; i < HEIGHT; ++i)
    {
        for (j = 0; j < WIDTH; ++j, ++x)
			if (map[x] < WALL_VER)
				mvaddch(i, j,  img[map[x]]);
    }
    attroff(COLOR_PAIR(2));
}

void play()
{
    game_active = 1;
    score = 0;
    direction pdir, pdir_next;
    pdir = pdir_next = NONE;
    byte i = 0;
    timeout(100);   // non-blocking getch() time
	while(score < dot_count)
	{
	    if (!game_active)
            return;
        switch (getch())
        {
            case KEY_LEFT : pdir_next = LEFT ; break;
            case KEY_UP   : pdir_next = UP   ; break;
            case KEY_RIGHT: pdir_next = RIGHT; break;
            case KEY_DOWN : pdir_next = DOWN ; break;
        }
        if (legal_dir(pac, pdir_next))
        {
            pdir = pdir_next;
        }
        switch (pdir)
        {
            case LEFT : pac_move(-1,  0); break;
            case UP   : pac_move( 0, -1); break;
            case RIGHT: pac_move( 1,  0); break;
            case DOWN : pac_move( 0,  1); break;
            default   : ;
        }
        draw_ch(pac);
        if (!(i++%40)) continue;
        en_move();
        mvprintw(2, WIDTH+1, " SCORE");
        mvprintw(3, WIDTH+1, " %5d", score);
	}

    attron(COLOR_PAIR(pac->col));
    mvprintw(HEIGHT/2-1, WIDTH/2-5, " Y O U");
    mvprintw(HEIGHT/2+1, WIDTH/2-5, "W I N !");
    attroff(COLOR_PAIR(pac->col));
    timeout(-1);
    game_active = 0;
}

void pac_move(byte x, byte y)
{
    x += pac->x;
    y += pac->y;
    if (legal_move(x, y))
    {
        if (map[y*WIDTH + x] == PATH_DOT) ++score;
        map[pac->y*WIDTH + pac->x] = map[y*WIDTH + x] = PATH_EMPTY;
        draw_trail(pac);
        pac->x = x;
        pac->y = y;
    }
}

inline void draw_trail(Game_Char* ch)
{
    attron(COLOR_PAIR(2));
    mvaddch(ch->y, ch->x, img[ch->trail]);
    attroff(COLOR_PAIR(2));
}

void en_move()
{
    byte i;
    for (i = 0; i < EN_MAX; ++i)
    {
        if (enemies[i] == NULL) continue;
        en_move_dir(enemies[i], en_move_decide(enemies[i]));
        draw_ch(enemies[i]);
        if (!game_active)
        {
            attron(COLOR_PAIR(enemies[i]->col));
            mvprintw(HEIGHT/2-1, WIDTH/2-5, " G A M E");
            mvprintw(HEIGHT/2+1, WIDTH/2-5, "O V E R !");
            attroff(COLOR_PAIR(enemies[i]->col));
            timeout(-1);
            return;
        }
    }
}

direction en_move_decide(Game_Char* en)
{
    char hor = pac->x - en->x;
    char ver = pac->y - en->y;

	direction primary, secondary;

    if (abs(hor)+abs(ver) <= 1)
	{
        game_active = 0;
	}

	if (((float)rand()/RAND_MAX) > 0.5)
	{
		primary = hor > 0 ? RIGHT:LEFT;
		secondary = ver > 0 ? DOWN:UP;
	}
	else
	{
		primary = ver > 0 ? DOWN:UP;
		secondary = hor > 0 ? RIGHT:LEFT;
	}
	if (legal_dir(en, primary) && not_backtracking(en, primary))
	{
		return primary;
	}
	else if (legal_dir(en, secondary) && not_backtracking(en, secondary))
	{
		return secondary;
	}
	else if (legal_dir(en, opp_dir(secondary)) && not_backtracking(en, opp_dir(secondary)))
	{
		return opp_dir(secondary);
	}
	else if (legal_dir(en, opp_dir(primary)) && not_backtracking(en, opp_dir(primary)))
	{
		return opp_dir(primary);
	}
	else
    {
        return en->opp_last_move;
    }
}

void en_move_dir(Game_Char* en, direction d)
{
    en->opp_last_move = opp_dir(d);
	switch (d)
	{
        case LEFT : en_move_vec(en, -1,  0); break;
        case UP   : en_move_vec(en,  0, -1); break;
        case RIGHT: en_move_vec(en,  1,  0); break;
        case DOWN : en_move_vec(en,  0,  1); break;
		default   : return;
	}
}

inline void en_move_vec(Game_Char* en, byte x, byte y)
{
    draw_trail(en);
	en->x += x;
	en->y += y;
	en->trail = map[en->y*WIDTH + en->x];
}

inline byte not_backtracking(Game_Char* en, direction d)
{
    return d != en->opp_last_move;
}

inline direction opp_dir(direction dir)
{
	switch (dir)
	{
		case LEFT: return RIGHT;
		case UP: return DOWN;
		case RIGHT: return LEFT;
		case DOWN: return UP;
		default: return NONE;
	}
}

void draw_ch(Game_Char* ch)
{
	attron(COLOR_PAIR(ch->col));
	mvaddch(ch->y, ch->x,  img[PAC] | A_ALTCHARSET);
	attroff(COLOR_PAIR(ch->col));
}

inline byte legal_move(byte x, byte y)
{
	return map[y*WIDTH + x] == PATH_EMPTY || map[y*WIDTH + x] == PATH_DOT;
}

inline byte legal_dir(Game_Char* ch, direction d)
{
	switch (d)
	{
		case LEFT : return legal_move(ch->x-1, ch->y  );
		case UP   : return legal_move(ch->x  , ch->y-1);
		case RIGHT: return legal_move(ch->x+1, ch->y  );
		case DOWN : return legal_move(ch->x  , ch->y+1);
		default   : return 0;
	}
}
