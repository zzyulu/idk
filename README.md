#include <iostream>
#include <thread>
#include <chrono>
#include <iomanip>
#include <ctime>
#include <vector>
#include <mutex>
#include "IntQueueHW6.h"
using namespace std;

IntQueueHW6 chairs(0);  //constructor is given as parametric only, will be changed in main
int number_of_players;

mutex mutexofthreads;

struct player{
    thread thread_of_player;
    int id;
    bool can_they_still_play = true;
};

vector <player *> player_pointers;

void threadofplayer(int id){

    this_thread::sleep_until(chrono::system_clock::now() + 2000ms);  //fairness

    // Begin: code taken from threads8.cpp
    time_t tt = chrono::system_clock::to_time_t (chrono::system_clock::now());  //gets the current time
    struct tm *ptm = new struct tm;  //creating the time struct to be used in thread
    localtime_s(ptm, &tt);  //converting the time structures
    // End: code taken from threads8.cpp

    mutexofthreads.lock();  //mutex is locked so that they can select in a proper way (no garbling up of chairs)
    if (chairs.isFull() == true){
        ostringstream output;  //output as ostringstream so that the output is shown as a single piece
        output << "Player "<< id << " couldn't capture a chair." << endl;
        cout << output.str();
        for (int i = 0; i < player_pointers.size(); i++){
            if (player_pointers[i]->thread_of_player.get_id() == this_thread::get_id()){  //if ids of threads are equal that means the thread belongs to that player
                player_pointers[i]->can_they_still_play = false; //this will be used to determine the eliminated player
            }
        }
        chairs.clear();  //chairs cleared for new round
        delete ptm; //avoiding any memory leaks
        ptm = nullptr;
        mutexofthreads.unlock();
        return;
    }
    else{
        chairs.enqueue(id);
        cout << "Player " << id << " captured a chair at " << put_time(ptm,"%X") << "." << endl;
        delete ptm;
        ptm = nullptr;
        mutexofthreads.unlock();
    }
}

int main() {
    cout << "Welcome to Musical Chairs game!"<<"\n"<<"Enter the number of players in the game:"<< endl;
    cin >> number_of_players;
    vector <player *> new_player_pointers(number_of_players);
    for (int i = 0; i < number_of_players; i++){
        new_player_pointers[i] = new player;
        new_player_pointers[i]->id = i;
    }
    player_pointers = new_player_pointers;  //since the player pointers vector is global, i needed a second vector to fill with players and then i equalized each vector
    cout << "Game Start!" << endl;
    while (true){
        IntQueueHW6 chairs_new(number_of_players-1);
        chairs = chairs_new;  //same thing with player_pointers
        // Begin: code taken from threads8.cpp
        time_t tt = chrono::system_clock::to_time_t (chrono::system_clock::now());  //gets the current time
        struct tm *ptm = new struct tm;  //creating the time struct to be used in thread
        localtime_s(ptm, &tt);  //converting the time structures
        cout << "Time is now " << put_time(ptm,"%X") << endl;
        // End : code taken from threads8.cpp

        for (int i = 0; i < player_pointers.size(); i++){
            player_pointers[i]->thread_of_player = thread(&threadofplayer, player_pointers[i]->id);  //id is the variable for threadofplayer func
        }
        for (int i = 0; i < player_pointers.size(); i++){
            player_pointers[i]->thread_of_player.join();  //after this for loop, threads will work in a random order
        }
        while (true){
            for (int i = 0; i < player_pointers.size();i++){
                if (player_pointers[i]->can_they_still_play == false){  //found the eliminated one
                    player * temp = player_pointers[i];
                    for (int x = i; x < player_pointers.size(); x++){ //removing the eliminated from player_pointers
                        player_pointers[x] = player_pointers[x+1];
                    }
                    player_pointers.pop_back(); //last one is a dummy pointer after the for loop
                    number_of_players -= 1;
                    delete temp;  //avoid memory leaks
                    temp = nullptr;
                    ostringstream remainder;  //i want to build the output string then show it as a single piece
                    remainder << "Remaining players are as follows: ";
                    for (int i = 0; i < player_pointers.size(); i++){  // adding the remainders into output
                        remainder << player_pointers[i]->id << " ";
                    }
                    remainder << endl;
                    cout << remainder.str();
                    break; //done with this loop
                }
            }
            break;
        }

        if (player_pointers.size() == 1){  //end of the game since 1 player remains
            cout << "Game Over!" << "\n" << "Winner is Player " << player_pointers[0]->id << "!";  //one element in player_pointers
            break;
        }
    }

    for (int i = 0; i < player_pointers.size(); i++){  //avoiding memory leaks
        delete player_pointers[i];
        player_pointers[i] = nullptr;
    }
    return 0;
}
