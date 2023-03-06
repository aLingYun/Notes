项目的树形结构：
```bash
.
├── main.cpp
├── Makefile
├── piano_key.cpp
├── piano_key.h
└── res
    ├── a1.wav
    ├── _A1.wav
    ...
```
`Makefile` 的内容：
```makefile
CC=g++
SRCS=main.cpp piano_key.cpp
CFLAGS=-g -lSDL2main -lSDL2 -lSDL2_image -lSDL2_mixer -lSDL2_net -lSDL2_ttf #-O2 #-mwindows

study.exe: SDL_Study.cpp
    $(CC) $(SRCS) $(CFLAGS) -o app
```
`piano_key.h` 的内容：
```C++
#ifndef PIANO_KEY_H
#define PIANO_KEY_H

#include <SDL2/SDL.h>

enum KEYCOLOR { black, white };

class PIANOKEY {
private:
  SDL_Rect key;

public:
  PIANOKEY();
  PIANOKEY(SDL_Rect kk);
  ~PIANOKEY();

  void showKey(SDL_Renderer *rend, KEYCOLOR key_color);
  SDL_Rect getKey();
  void setKey(SDL_Rect kk);
};
#endif // !PIANO_KEY_H
```
`piano_key.h` 的内容：
```c++
#include "piano_key.h"
#include <SDL2/SDL_blendmode.h>
#include <SDL2/SDL_render.h>

PIANOKEY::PIANOKEY() {}

PIANOKEY::PIANOKEY(SDL_Rect kk) : key(kk) {}

PIANOKEY::~PIANOKEY() {}

void PIANOKEY::showKey(SDL_Renderer *rend, KEYCOLOR key_color) {
  SDL_SetRenderDrawBlendMode(rend, SDL_BLENDMODE_BLEND);
  if (key_color == white) {
    SDL_SetRenderDrawColor(rend, 255, 255, 255, 180);
  } else {
    SDL_SetRenderDrawColor(rend, 0, 0, 0, 255);
  }
  SDL_RenderFillRect(rend, &key);
  SDL_SetRenderDrawColor(rend, 0, 0, 0, 255);
  SDL_RenderDrawRect(rend, &key);
}

void PIANOKEY::setKey(SDL_Rect kk) { key = kk; }

SDL_Rect PIANOKEY::getKey() { return key; }
```
`main.cpp` 的内容：
```c++
#include "piano_key.h"
#include <SDL2/SDL.h>
#include <SDL2/SDL_events.h>
#include <SDL2/SDL_image.h>
#include <SDL2/SDL_mixer.h>
#include <SDL2/SDL_render.h>
#include <SDL2/SDL_video.h>

#define PC 0
#define AD 1
#define PLATFORM PC

#define WHITEKEYNUM 18
#define BLACKKEYNUM 13

#define DISPLAY PIANOW / 2
#define WHITEKEYW (PIANOW - DISPLAY)
#define WHITEKEYH (PIANOH / WHITEKEYNUM)
#define BLACKKEYW (WHITEKEYW * 2 / 3)
#define BLACKKEYH (WHITEKEYH * 2 / 3)

const char *blackWav[BLACKKEYNUM] = {
    "./res/f#.wav",  "./res/g#.wav",  "./res/a#.wav",  "./res/c#.wav",
    "./res/d#.wav",  "./res/f1#.wav", "./res/g1#.wav", "./res/a1#.wav",
    "./res/c2#.wav", "./res/d2#.wav", "./res/f2#.wav", "./res/g2#.wav",
    "./res/a2#.wav"};

const char *whiteWav[WHITEKEYNUM] = {
    "./res/f.wav",  "./res/g.wav",  "./res/a.wav",  "./res/b.wav",
    "./res/c1.wav", "./res/d1.wav", "./res/e1.wav", "./res/f1.wav",
    "./res/g1.wav", "./res/a1.wav", "./res/b1.wav", "./res/c2.wav",
    "./res/d2.wav", "./res/e2.wav", "./res/f2.wav", "./res/g2.wav",
    "./res/a2.wav", "./res/b2.wav"};

bool boPointInRect(SDL_Point point, SDL_Rect rect) {
  if ((point.x >= rect.x) && (point.x <= rect.x + rect.w) &&
      (point.y >= rect.y) && (point.y <= rect.y + rect.h)) {
    return true;
  }
  return false;
}

int main(int argc, char *argv[]) {
  int PIANOW;
  int PIANOH;
  SDL_Init(SDL_INIT_EVERYTHING);
  Mix_OpenAudio(22050, MIX_DEFAULT_FORMAT, 2, 4096);
  IMG_Init(IMG_INIT_JPG);
  SDL_Window *window =
      SDL_CreateWindow("piano", SDL_WINDOWPOS_CENTERED, SDL_WINDOWPOS_CENTERED,
                       216, 468, SDL_WINDOW_SHOWN);
  SDL_Renderer *renderer =
      SDL_CreateRenderer(window, -1, SDL_RENDERER_ACCELERATED);
  SDL_Surface *background = IMG_Load("./res/background.jpg");
  SDL_Texture *text = SDL_CreateTextureFromSurface(renderer, background);

  bool quit = false;
  SDL_Event event;
  SDL_Point point;
  Mix_Music *scratch = NULL;
  PIANOKEY keyWhite[WHITEKEYNUM];
  PIANOKEY keyBlack[BLACKKEYNUM];

  SDL_RenderClear(renderer);
  SDL_RenderCopy(renderer, text, NULL, NULL);

  SDL_GetWindowSize(window, &PIANOW, &PIANOH);

  for (int i = 0; i < WHITEKEYNUM; i++) {
    keyWhite[i].setKey({0, i * WHITEKEYH, WHITEKEYW, WHITEKEYH - 1});
    keyWhite[i].showKey(renderer, white);
  }
  for (int i = 0, j = 0; (i < WHITEKEYNUM) && (j < BLACKKEYNUM); i++) {
    if ((i == 3) || (i == 6) || (i == 10) || (i == 13)) {
      continue;
    } else {
      keyBlack[j].setKey({WHITEKEYW - BLACKKEYW,
                          i * WHITEKEYH + WHITEKEYH - BLACKKEYH / 2, BLACKKEYW,
                          BLACKKEYH});
      keyBlack[j].showKey(renderer, black);
      j++;
    }
  }

  SDL_RenderPresent(renderer);
  while (!quit) {
    while (SDL_PollEvent(&event)) {
      switch (event.type) {
      case SDL_QUIT:
        quit = true;
        break;
#if PLATFORM == PC
      case SDL_MOUSEBUTTONDOWN:
        if (event.button.button == SDL_BUTTON_LEFT) {
          point.x = event.button.x;
          point.y = event.button.y;
        }
        break;
#elif PLATFORM == AD
      case SDL_MOUSEBUTTONDOWN:
        if (event.button.button == SDL_FINGERDOWN) {
          point.x = event.button.x;
          point.y = event.button.y;
        }
        break;
#endif
      default:
        break;
      }
    }
    for (int i = 0; i < WHITEKEYNUM; i++) {
      SDL_Rect WhiteRectTemp = keyWhite[i].getKey();
      SDL_Rect BlackRectTemp = keyBlack[i].getKey();
      if (i < BLACKKEYNUM) {
        if (boPointInRect(point, BlackRectTemp)) {
          Mix_HaltMusic();
          scratch = Mix_LoadMUS(blackWav[i]);
          Mix_PlayMusic(scratch, 1);
          break;
        }
        if (boPointInRect(point, WhiteRectTemp)) {
          Mix_HaltMusic();
          scratch = Mix_LoadMUS(whiteWav[i]);
          Mix_PlayMusic(scratch, 1);
          break;
        }
      } else {
        if (boPointInRect(point, WhiteRectTemp)) {
          Mix_HaltMusic();
          scratch = Mix_LoadMUS(whiteWav[i]);
          Mix_PlayMusic(scratch, 1);
          break;
        }
      }
    }
    // SDL_RenderPresent(renderer);
    point = {PIANOW, PIANOH};
    SDL_Delay(1000 / 60);
  }
  SDL_RenderClear(renderer);
  Mix_FreeMusic(scratch);
  Mix_CloseAudio();
  SDL_DestroyWindow(window);
  SDL_DestroyRenderer(renderer);
  SDL_Quit();
  return 0;
}
```