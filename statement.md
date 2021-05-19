# About

C++ starter code for the CG Spring Challenge 2021. (Totoro / Photosynthesis / Whatever you want to call it)

Algorithm is Chokudai's variant of Beam Search.


Totoro has a lot of high ranking bots with just basic heuristics (if/else conditions), therefore a basic search with some decent eval will get you
very far on the leaderboard. Best way to do this is by analysing the game and spot best plays. Doing this will then get you accustomed to the code.

You're meant to improve on the code and optimize it yourself. Bitsets are a personal preference.

Beam Search is about taking n most promissing moves and exploring their future outcomes. This variant however has n (the beam width) starting at 1 
and then gradually increase with each iteration.

# Code


```C++ runnable
#pragma GCC optimize "O3,mavx2,omit-frame-pointer,inline,unroll-loops"
#include <iostream>
#include <string>
#include <vector>
#include <algorithm>
#include <chrono>
#include <map>
#include <cmath> 
#include <bitset>
#include <set>
#include <cstring>
#include <iomanip> 

using namespace std;
using namespace chrono;

// If you liked the starter, please have at least one message containing a JoJo reference.
// Thank you for your appreciation!
vector<string> greetings = {" Omae wa mou shindeiru",  // Reference: Fist of the North Star 
    " May your journey overflow with curses and blessings.", // Reference: Made in Abyss
    " Opponent eval: 1ms, score: MUDA"}; // Reference: JoJo's Bizarre Adventures

int turn = 0;

bitset<37> r0; // tiles with 0 richness

struct Node {
    int tile;
    array<Node*,6> nhex; // neighbours
    array<array<bitset<37>,3>,6> shadow; // precomputed shadows of various tree levels

    int r; // richness for this Tile

    Node(int t1) {
        tile = t1;
    }
};
array<Node*,37> mapp;

void initialize(int x2) {
    for (int dir = 0; dir < 6; dir++) {
        if (mapp[x2]->nhex[dir] != NULL) {
            mapp[x2]->shadow[dir][0][mapp[x2]->nhex[dir]->tile] = 1;

            if (mapp[x2]->nhex[dir]->nhex[dir] != NULL) {
                mapp[x2]->shadow[dir][1][mapp[x2]->nhex[dir]->nhex[dir]->tile] = 1;
                
                if (mapp[x2]->nhex[dir]->nhex[dir]->nhex[dir] != NULL) {
                    mapp[x2]->shadow[dir][2][mapp[x2]->nhex[dir]->nhex[dir]->nhex[dir]->tile] = 1;
                }
            }
        }
    }
}

struct GameState {
    int day, nutrients, sun, score, wait, sun2, score2, wait2;
    bitset<37> seeds;
    bitset<37> tree1;
    bitset<37> tree2;
    bitset<37> tree3;
    bitset<37> own;
    bitset<37> op;
    bitset<37> dormant;
    bitset<37> shadows1;
    bitset<37> shadows2;
    bitset<37> shadows3;
    vector<GameState*> future;
    int action;
    
    GameState() {}

    void update(int idx, int lvl, int isMine, int isDormant) {
        seeds[idx] = (lvl == 0);
        tree1[idx] = (lvl == 1);
        tree2[idx] = (lvl == 2);
        tree3[idx] = (lvl == 3);
        own[idx] = isMine;
        op[idx] = !isMine;
        dormant[idx] = isDormant;
    }

    // when the day passes sun gain is computed
    void count_sun() {
        sun += (tree1 & own & ~(shadows1 | shadows2 | shadows3) ).count() 
            + (tree2 & own & ~(shadows2 | shadows3) ).count() * 2 
            + (tree3 & own & ~(shadows3) ).count() * 3;
    }

    void recast_all_shadows() {
        int dir = day % 6;
        shadows1.reset();
        shadows2.reset();
        shadows3.reset();

        for (int nidx = 0; nidx < 37; nidx++) {
            if (tree1[nidx]) shadows1 |= mapp[nidx]->shadow[dir][0];
            if (tree2[nidx]) shadows2 |= mapp[nidx]->shadow[dir][1];
            if (tree3[nidx]) shadows3 |= mapp[nidx]->shadow[dir][2];
        }

        count_sun();
    }

    void copy_(GameState *g) {
        day = g->day;
        nutrients = g->nutrients;
        sun = g->sun;
        score = g->score;
        wait = g->wait;
        sun2 = g->sun2;
        score2 = g->score2;
        wait2 = g->wait2;
        seeds = g->seeds;
        tree1 = g->tree1;
        tree2 = g->tree2;
        tree3 = g->tree3;
        own = g->own;
        op = g->op;
        dormant = g->dormant;
        action = g->action;
    }
};

vector<string> actions;

int main() {
    int nCells; // 37 always !
    cin >> nCells; cin.ignore();

    // create hex grid
    for (int i = 0; i < 37; i++) {
        Node *n = new Node(i);
        mapp[i] = n;
    }

    // connect neighbours
    for (int i = 0; i < 37; i++) {
        int index, neigh; // i == index always !
        cin >> index >> mapp[i]->r; cin.ignore(); 
        r0[i] = (mapp[i]->r == 0);

        for (int j = 0; j < 6; j++) {
            cin >> neigh; cin.ignore();
            mapp[i]->nhex[j] = (neigh != -1) ? mapp[neigh] : NULL;
        }
    }

    // shadows
    for (int i = 0; i < 37; i++) {
        initialize(i);
        for (int j = 0; j < 6; j++) {
            mapp[i]->shadow[j][1] |= mapp[i]->shadow[j][0];
            mapp[i]->shadow[j][2] |= mapp[i]->shadow[j][1];
        }
    }

    while (1) {
        GameState* g = new GameState();

        int nTrees;
        cin >> g->day >> g->nutrients >> g->sun >> g->score >> g->sun2 >> g->score2 >> g->wait2 >> nTrees; cin.ignore();
        
        high_resolution_clock::time_point start = high_resolution_clock::now();

        turn++;
        cerr << "Turn " << turn << " Day " << g->day << endl;

        for (int i = 0; i < nTrees; i++) {
            int cellIndex, size, isMine,  isDormant;
            cin >> cellIndex >> size >> isMine >> isDormant; cin.ignore();
            g->update(cellIndex,size,isMine,isDormant);
        }

        actions.clear();

        int nMoves;
        cin >> nMoves; cin.ignore();

        bitset<37> tree_spots = (g->tree1 | g->tree2 | g->tree3) & g->own;
        bitset<37> shadow_mask = tree_spots;
        for (int i = 0; i < 37; i++)
            for (int j = 0; j < 6; j++)
                if (tree_spots[i])
                    shadow_mask |= mapp[i]->shadow[j][2];

        // process moves and store them into the initial state
        for (int i = 0; i < nMoves; i++) {
            string s;
            getline(cin, s);
            actions.push_back(s);
            if (s[0] == 'W') {
                GameState* gp = new GameState();
                gp->copy_(g);
                gp->day++;
                gp->recast_all_shadows();
                gp->dormant.reset();
                gp->action = i;
                g->future.push_back(gp);
            } else if (s[0] == 'S' && (g->seeds & g->own).count() <= 0) {
                string x = s.substr(s.find(' ') + 1);
                size_t pos = x.find(' ');
                string token = x.substr(0, pos);
                int idx = stoi(token);
                x.erase(0, pos + 1);
                int idx2 = stoi(x);

                GameState* gp = new GameState();
                gp->copy_(g);
                gp->seeds[idx2] = 1;
                gp->sun = g->sun - (g->seeds & g->own).count();
                gp->dormant[idx] = 1;
                gp->dormant[idx2] = 1;
                gp->own[idx2] = 1;
                gp->action = i;
                g->future.push_back(gp);
            } else if (s[0] == 'G') {
                string x = s.substr(s.find(' ') + 1);
                int idx = stoi(x);

                GameState* gp = new GameState();
                gp->copy_(g);
                gp->dormant[idx] = 1;
                if (g->seeds[idx]) {
                    gp->seeds[idx] = 0;
                    gp->tree1[idx] = 1;
                    gp->sun -= (g->tree1 & g->own).count() + 1;
                } else if (g->tree1[idx]) {
                    gp->tree1[idx] = 0;
                    gp->tree2[idx] = 1;
                    gp->sun -= (g->tree2 & g->own).count() + 3;
                } else if (g->tree2[idx]) {
                    gp->tree2[idx] = 0;
                    gp->tree3[idx] = 1;
                    gp->sun -= (g->tree3 & g->own).count() + 7;
                }
                gp->dormant[idx] = 1;
                gp->action = i;
                g->future.push_back(gp);
            } else if (s[0] == 'C') {
                string x = s.substr(s.find(' ') + 1);
                int idx = stoi(x);

                GameState* gp = new GameState();
                gp->copy_(g);
                gp->sun += - 4;
                gp->score += mapp[idx]->r + g->nutrients;
                gp->nutrients--;
                gp->tree3[idx] = 0;
                gp->own[idx] = 0;
                gp->action = i;
                g->future.push_back(gp);
            }
        }

        double total = duration<double, std::milli>(high_resolution_clock::now () - start).count();
        cerr << setprecision(5) << "Turn: " << turn << " Time: " << total << endl;
        
        GameState* GrandState = g->future[0];

        array<vector<GameState*>,201> AllStates;
        for (int i = 0; i < g->future.size(); i++)
            AllStates[1].push_back(g->future[i]);
        
        int width = 1;
        int maxTime = turn == 1 ? 50 : 5;

        int current_time = duration<double,std::milli>(high_resolution_clock::now()-start).count();

        // start the search algo
        while (duration<double,std::milli>(high_resolution_clock::now()-start).count() < 20) {
            for (int t = 0; t < 200; t++) {
                for (int i = 0; i < width; i++) {
                    if (AllStates[t].empty()) break;

                    std::sort(std::begin(AllStates[t]), std::end(AllStates[t]),
                        [](const GameState* lhs, const GameState* rhs) -> bool {
                            return lhs->seeds.count() < rhs->seeds.count();
                        });

                    GameState *NowState = AllStates[t][AllStates[t].size()-1];
                    AllStates[t].pop_back();

                    if (NowState->day > 23) {
                        if (NowState->seeds.count() > GrandState->seeds.count())
                            GrandState = NowState;
                        continue;
                    }

                    // WAIT
                    if (NowState->day <= 23) {
                        GameState* gp = new GameState();
                        gp->copy_(NowState);
                        gp->day = NowState->day+1;
                        gp->recast_all_shadows();
                        gp->dormant.reset();
                        AllStates[t+1].push_back(gp);
                    }

                    // SEED
                    bitset<37> tree_spots = (NowState->tree1 | NowState->tree2 | NowState->tree3) & NowState->own & ~NowState->dormant;
                    if (NowState->seeds.count() <= NowState->sun) {
                        bitset<37> shadow_mask = tree_spots;
                        for (int k = 0; k < 37; k++) {
                            for (int j = 0; j < 6; j++) {
                                if (tree_spots[k])
                                    shadow_mask |= mapp[k]->shadow[j][2];
                            }
                        }

                        bitset<37> occupied = NowState->seeds & NowState->tree1 & NowState->tree2 & NowState->tree3;
                        if (tree_spots.count() > 0)
                        for (int j = tree_spots._Find_first(); j < tree_spots.size(); j = tree_spots._Find_next(j)) {
                            int lvl = NowState->tree3[j] ? 3 : NowState->tree2[j] ? 2 : NowState->tree1[j] ? 1 : 0;
                            bitset<37>seed_spots = ~occupied & ~r0;
                            for (int k = seed_spots._Find_first(); k < seed_spots.size(); k = seed_spots._Find_next(k)) {
                                GameState* gp = new GameState();
                                gp->copy_(NowState);
                                gp->seeds[k] = 1;
                                gp->sun -= NowState->seeds.count();
                                gp->dormant[j] = 1;
                                gp->dormant[k] = 1;
                                gp->own[k] = 1;
                                AllStates[t+1].push_back(gp);
                            }
                        }
                    }

                    // GROW
                    tree_spots = (NowState->seeds | NowState->tree1 | NowState->tree2 | NowState->tree3) & NowState->own & ~NowState->dormant;
                    if (tree_spots.count() > 0)
                    for (int j = tree_spots._Find_first(); j < tree_spots.size(); j = tree_spots._Find_next(j)) {
                        if (NowState->seeds[j] && (NowState->tree1 & NowState->own).count() + 1 < NowState->sun) {
                            GameState* gp = new GameState();
                            gp->copy_(NowState);
                            gp->seeds[j] = 0;
                            gp->tree1[j] = 1;
                            gp->sun -= (NowState->tree1 & NowState->own).count() + 1;
                            gp->dormant[j] = 1;
                            AllStates[t+1].push_back(gp);
                        } else if (NowState->tree1[j] && (NowState->tree2 & NowState->own).count() + 3 < NowState->sun) {
                            GameState* gp = new GameState();
                            gp->copy_(NowState);
                            gp->tree1[j] = 0;
                            gp->tree2[j] = 1;
                            gp->sun -= (NowState->tree2 & NowState->own).count() + 3;
                            gp->dormant[j] = 1;
                            AllStates[t+1].push_back(gp);
                        } else if (NowState->tree2[j] && (NowState->tree3 & NowState->own).count() + 7 > NowState->sun) {
                            GameState* gp = new GameState();
                            gp->copy_(NowState);
                            gp->tree2[j] = 0;
                            gp->tree3[j] = 1;
                            gp->sun -= (NowState->tree3 & NowState->own).count() + 7;
                            gp->dormant[j] = 1;                    
                            AllStates[t+1].push_back(gp);
                        }
                    }

                    // COMPLETE
                    if (4 <= NowState->sun) {
                        tree_spots = NowState->tree3 & NowState->own & ~NowState->dormant;
                        if (tree_spots.count() > 0)
                        for (int j = tree_spots._Find_first(); j < tree_spots.size(); j = tree_spots._Find_next(j)) {
                            GameState* gp = new GameState();
                            gp->copy_(NowState);
                            gp->tree3[j] = 0;
                            gp->sun -= 4;
                            gp->own[j] = 0;
                            gp->score += mapp[j]->r + gp->nutrients;
                            gp->nutrients--;
                            AllStates[t+1].push_back(gp);
                        }
                    }
                    
                }
            }

            width++;
        }

        total = duration<double, std::milli>(high_resolution_clock::now () - start).count();
        cerr << setprecision(5) << "Extra Time: " << total << endl;

        cout << actions[ GrandState->action ] <<
        greetings[rand()%greetings.size()] << endl;
    }
}
```

# TODO

- Fix the sorting function
- Write the eval / scoring
- Optimize the code. For example a Heap could do a lot more than a vector for states or look up zobrist hash for copying gamestates.
- Reuse states next turn.
- Improve by adding opponent prediction or a separate search
- Fix the time limit conditions.
- You can precompute a lot more than just shadows (see the `initialize()` function). For example distances, tree costs and much more.
- Further improve based on strategies listed on the forums:
https://www.codingame.com/forum/t/spring-challenge-2021-feedbacks-strategies/190849


# Feedback and Suggestions

are very much appreciated. Looking forward to hear from you.

Thanks!

# 
~ AntiSquid
