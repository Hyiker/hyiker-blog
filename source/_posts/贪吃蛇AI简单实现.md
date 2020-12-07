---
title: 贪吃蛇AI简单实现
date: 2020-03-31 22:28:24
tags: AI 贪吃蛇 小游戏
keywords: AI 贪吃蛇 小游戏
categories: AI
description: 随手写的小玩意
summary_img:
---

使用决策树模型实现了一个简单的智障寻路。

<!-- more -->

```cpp
#include <bits/stdc++.h>
#include <chrono>
#define PLAYGROUND_WIDTH 8
using std::vector, std::ostream, std::string, std::queue, std::set;
vector<string> fname{"up", "left", "right", "down"};
enum BlockType { empty, snake, sugar, wall, head };
enum Forward { up = 0, left = 1, right = 2, down = 3 };

ostream &operator<<(ostream &os, Forward f) {
    if (f == up) {
        os << "up";
    } else if (f == left) {
        os << "left";
    } else if (f == right) {
        os << "right";
    } else if (f == down) {
        os << "down";
    } else {
        os << "unknown";
    }
    return os;
}
class Playground;
class Snake {
  private:
    /* 用于保存身体的链表 */
    vector<int *> body;
    int *head{nullptr};
    int length;
    Playground *pg{nullptr};

  public:
    Forward forward;
    Snake() {
        int *a = new int[2]{3, 4};
        int *b = new int[2]{4, 4};
        body.push_back(a);
        body.push_back(b);
        head = body[0];
        length = 2;
    }
    Snake(Snake *snake, Playground *pg) {
        this->pg = pg;
        length = snake->length;
        vector<int *> sb = snake->getBody();
        for (int i = 0; i < sb.size(); i++) {
            int *node = new int[2]{sb[i][0], sb[i][1]};
            body.push_back(node);
        }
        head = body[0];
    }
    vector<int *> getBody() { return this->body; }
    int *getHead() { return this->head; }
    void setPlayground(Playground *pg) { this->pg = pg; }
    void move(Forward);
    void move();
    bool isDead() {
        if (head[0] == 0 || head[0] == PLAYGROUND_WIDTH - 1 || head[1] == 0 ||
            head[1] == PLAYGROUND_WIDTH - 1) {
            return true;
        }

        for (int i = 1; i < length; i++) {
            if (body[i][0] == head[0] && body[i][1] == head[1]) {
                return true;
            }
        }
        return false;
    }
    int len() { return length; }
};
class Playground {
  private:
    Snake *_snake{nullptr};
    BlockType blocks[PLAYGROUND_WIDTH][PLAYGROUND_WIDTH]{empty};

  public:
    vector<int *> sugars;

    BlockType *operator[](const int i) { return blocks[i]; }
    friend ostream &operator<<(ostream &os, Playground &pg) {
        string res = "";
        BlockType type;
        pg.refresh();
        pg.generateSnake();
        pg.generateSugar();
        for (size_t i = 0; i < PLAYGROUND_WIDTH; i++) {
            for (size_t j = 0; j < PLAYGROUND_WIDTH; j++) {
                type = pg[i][j];
                if (type == empty) {
                    res += " ";
                } else if (type == wall) {
                    res += "*";
                } else if (type == snake) {
                    res += "□";
                } else if (type == sugar) {
                    res += "#";
                } else if (type == head) {
                    res += "■";
                }
                res += ' ';
            }
            res += '\n';
        }
        os << res;
        return os;
    }
    Playground() {
        for (int i = 0; i < PLAYGROUND_WIDTH; i++) {
            for (int j = 0; j < PLAYGROUND_WIDTH; j++) {
                blocks[i][j] = wall;
            }
        }
        refresh();
    }
    Playground(Playground &anc) {
        memcpy(blocks, anc.blocks,
               sizeof(BlockType) * PLAYGROUND_WIDTH * PLAYGROUND_WIDTH);
        for (int i = 0; i < anc.sugars.size(); i++) {
            sugars.push_back(new int[2]{anc.sugars[i][0], anc.sugars[i][1]});
        }

        this->_snake = new Snake(anc.getSnake(), this);
    }
    Playground(Snake *snake_) : Playground() { setSnake(snake_); }

    void setSnake(Snake *snake_) {
        _snake = snake_;
        snake_->setPlayground(this);
    }
    void generateSnake() {
        vector<int *> snake_body = _snake->getBody();
        int *_head = _snake->getHead();
        for (auto it : snake_body) {
            blocks[it[0]][it[1]] = snake;
        }
        blocks[_head[0]][_head[1]] = head;
    }
    void makeSugar(int count = 1) {
        for (int i = 0; i < count; i++) {
            int m, n;
            do {
                m = rand() % (PLAYGROUND_WIDTH);
                n = rand() % (PLAYGROUND_WIDTH);
            } while (blocks[m][n] != empty);
            int *su = new int[2]{m, n};
            sugars.push_back(su);
        }
    }
    void generateSugar() {
        for (auto it : sugars) {
            blocks[it[0]][it[1]] = sugar;
        }
    }
    void refresh() {
        for (int i = 1; i < PLAYGROUND_WIDTH - 1; i++) {
            for (int j = 1; j < PLAYGROUND_WIDTH - 1; j++) {
                blocks[i][j] = empty;
            }
        }
    }
    Snake *getSnake() { return _snake; }
    bool hasEnd() { return _snake->isDead(); }
    bool toTail() {
        vector<int *> body = _snake->getBody();
        vector<int> ext_lst;
        queue<int *> q;

        int *tail = body.back();
        int *head = _snake->getHead();
        int *tmp{nullptr};

        q.push(head);

        while (q.size() > 0) {
            tmp = q.front();
            q.pop();
            int neighbors[4][2]{{tmp[0], tmp[1] - 1},
                                {tmp[0] - 1, tmp[1]},
                                {tmp[0] + 1, tmp[1]},
                                {tmp[0], tmp[1] + 1}};
            for (auto neighbor : neighbors) {
                if (neighbor[0] < 1 || neighbor[0] >= PLAYGROUND_WIDTH - 1 ||
                    neighbor[1] < 1 || neighbor[1] >= PLAYGROUND_WIDTH - 1 ||
                    find(ext_lst.begin(), ext_lst.end(),
                         neighbor[0] * 100 + neighbor[1]) != ext_lst.end()) {
                    continue;
                }
                // std::cout << neighbor[0] << "," << neighbor[1] << std::endl;
                if (neighbor[0] == tail[0] && neighbor[1] == tail[1]) {
                    return true;
                } else if (blocks[neighbor[0]][neighbor[1]] != empty) {
                    continue;
                }
                ext_lst.push_back(neighbor[0] * 100 + neighbor[1]);
                q.push(neighbor);
            }
        }
        return false;
    }
};
bool directionAvailable(Forward c, Forward f) {
    return !((c == up && f == down) || (c == left && f == right) ||
             (c == right && f == left) || (c == down && f == up));
}
class Predictor {
  public:
    Forward forwards[4]{up, left, right, down};
    int judge(Playground pg, int score, int depth) {
        if (pg.hasEnd()) {
            return -10000000;
        }
        Snake snake = *pg.getSnake();
        score += 16 * snake.len() + 8 * pg.toTail();
        // score *= depth + 1;
        return score;
    }
    Forward best(Playground &pg, int depth = 3) {
        Forward now = pg.getSnake()->forward;
        int largest{-1000}, idx{0}, tmp;
        for (int i = 0; i < 4; i++) {
            if (!directionAvailable(now, forwards[i])) {
                continue;
            }
            // std::cout << now << "," << forwards[i] << std::endl;
            Playground npg = *new Playground(pg);
            npg.getSnake()->move(forwards[i]);
            tmp = pred(npg, depth);
            idx = tmp > largest ? i : idx;
            largest = tmp > largest ? tmp : largest;
        }
        while (idx + now == 3) {
            idx = rand() % 4;
        }
        return forwards[idx];
    }
    int pred(Playground &pg, int depth = 1, int score = 0) {
        if (depth == 0 || pg.hasEnd()) {
            return score;
        }
        int j = judge(pg, score, depth);
        int largest{0}, tmp;
        Forward now = pg.getSnake()->forward;
        for (int i = 0; i < 4; i++) {
            if (!directionAvailable(now, forwards[i])) {
                continue;
            }
            Playground npg = *new Playground(pg);
            npg.getSnake()->move(forwards[i]);
            tmp = pred(npg, depth - 1, j);
            largest = tmp > largest ? tmp : largest;
        }
        return largest;
    }
};

void Snake::move() {
    bool added = false;
    int *tail = new int[2]{body[body.size() - 1][0], body[body.size() - 1][1]};
    for (int i = body.size() - 1; i > 0; i--) {
        body[i][0] = body[i - 1][0];
        body[i][1] = body[i - 1][1];
    }
    if (forward == up) {
        head[0] -= 1;
    } else if (forward == left) {
        head[1] -= 1;
    } else if (forward == right) {
        head[1] += 1;
    } else if (forward == down) {
        head[0] += 1;
    }
    for (int i = 0; i < pg->sugars.size(); i++) {
        if (head[0] == pg->sugars[i][0] && head[1] == pg->sugars[i][1]) {
            pg->sugars.erase(pg->sugars.begin() + i);
            body.push_back(tail);
            added = true;
            break;
        }
    }
    if (!added) {
        delete[] tail;
        tail = nullptr;
    } else {
        length++;
        pg->makeSugar();
    }
}
void Snake::move(Forward f) {
    forward = f;
    move();
}

int main() {
    Snake snake;
    Playground pg(&snake);
    pg.makeSugar();
    Predictor p;
    Forward f;
    snake.move(up);
    srand(20011209);
    for (size_t i = 0; i < 2000; i++) {
        f = p.best(pg, 9);
        std::cout << pg << std::endl;
        snake.move(f);
        if (pg.hasEnd()) {
            std::cout << "GAME OVER" << std::endl;
            break;
        }
        std::this_thread::sleep_for(std::chrono::milliseconds(50));
    }
}

```
