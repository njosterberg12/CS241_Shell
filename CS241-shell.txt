/************************************************************
 * Id: oste1799
 * Purpose:
 *    Illustrate the concepts necessary to deal with the
 *    creation, demonstration, and deletion of background
 *    processes.
 *    A simple shell.
 * Compile:  gcc -g -Wall -o ttshell ttshell.c
 * Usage:    ./ttshell
 *************************************************************/
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <ctype.h>
#include <unistd.h>
#include <signal.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <fcntl.h>

#define MAX_WORDS 20
#define MAX_CMD 512
#define MAX_BG 5
#define INVALID_PID 0
#define SIGTERM 15
typedef struct {
   int pid; // If pid == INVALID_PID this is a free slot
   char command[MAX_CMD]; // The command the user entered
} bgjob_t;
int asArrayOfSubstrings(char *cmd, char **argv);
void forkExecution(char**cmdAsArray);
void help();
void bgFunc(bgjob_t bglist[],char **cmdAsArray);
void bglistFunc(bgjob_t bglist[]);
void bgkillFunc(bgjob_t bglist[], char **cmdAsArray);
void clearList(bgjob_t bglist[]);
int main (void)
{
   char cmd[MAX_CMD];
   char *cmdAsArray[MAX_WORDS];
   bgjob_t bglist[MAX_BG];
   for(int i;i<MAX_BG;i++)
   {
      bglist[i].pid = INVALID_PID;
   }
   int args = 0;
   printf("\nCreated By: Toby Tabladillo\nType \"help\" to see implemented commands.\n\n");
   for(;;) { //Loop forever until user enters "exit"
      printf("ttshell> "); // Print prompt
      fgets(cmd, MAX_CMD, stdin);
      if(cmd == NULL){
         printf("Unrecognized Command\n");
      }
      args = asArrayOfSubstrings(cmd, cmdAsArray); // Break the command into words
      cmdAsArray[args] = '\0';
      //the blank case
      if(cmdAsArray[0] == '\0'){
         //do nothing
      }
      else if(strcmp(cmdAsArray[0],"exit") == 0){
         // Do any clean up
         exit(0);
      }
      else if(strcmp(cmdAsArray[0],"help") == 0){
         //Display available user commands
         help();
      }
      else if(strcmp(cmdAsArray[0],"bg") == 0){
         bgFunc(bglist,&cmdAsArray[1]);
      }
      else if(strcmp(cmdAsArray[0],"bglist") == 0){
         bglistFunc(bglist);
      }
      else if(strcmp(cmdAsArray[0],"bgkill") == 0){
         bgkillFunc(bglist,&cmdAsArray[1]);
      }
      else{
         //Provide availability for running Unix commands
         forkExecution(cmdAsArray);
      }
   }// end for loop
}

//  PROC: asArrayOfSubstrings
//  Inp:  cmd: The original string
//  Outp: argv: The array
//        int: The nubmer of arguments
//  Desc: Assignment: Part 1
//        Tokeniszes cmd into an array of substings argv on space
int asArrayOfSubstrings(char *cmd, char **argv) {
   char* token = NULL;
   int argC = 0;
   char cmdCopy[MAX_CMD];
   strcpy(cmdCopy, cmd);
   token = strtok(cmdCopy, " ");
   while(token) { // tokenize cmp into the array argv
      argv[argC] = (char *)malloc(sizeof(char) * (strlen(token) + 1));
      if(token[strlen(token)-1]== '\n')
      {
         token[strlen(token)-1] = 0; // NULL terminate the string
      }
      strncpy(argv[argC], token, strlen(token) + 1);
      token = strtok(NULL, " ");
      argC++;
   }
   return argC;
}
/***help***/
//Provides a list that shows available commands
void help(){
   //Display menu of commands
   printf("pwd  - Print the working directory\n");
   printf("exit - Exit shell\n");
   printf("help - Display this message\n\n");
   printf("bglist         - List background programs\n");
   printf("bgkill <int>   - List background programs\n");
   printf("<UNIX cmd>     - List background programs\n");
   printf("bg <UNIX cmd>  - List background programs\n\n");
}
/***bgFunc***/
//Sends a specified task to the background (stored in bglist)
void bgFunc(bgjob_t bglist[], char **cmdAsArray){
   // Clear completed processes
   clearList(bglist);   
   // Find next available free slot
   int pid;
   int index = 0;
   while (index < MAX_BG){
      if(bglist[index].pid != INVALID_PID)  //If not free, move to next slot
         index++;
      else  //If free, break out of loop
         break;
      if(index == 4){  //All processes full, so tell the user and exit
         printf("There are no background process slots available.\n");
         //Do not allow the user to continue in the bg command
         exit(1);
      }
   }   
   //Free slot found, so create parent and child processes
   pid = fork();
   if(pid < 0)
   {
      printf("The fork failed\n");
      exit(1);
   }
   if (pid == 0)  //Child process
   {
      int stdin_fd = open("/dev/null", O_WRONLY);
      if (stdin_fd == -1)
      {
         _exit(127);
      }
      dup2(stdin_fd, STDOUT_FILENO);
      close(stdin_fd);
      execvp(*cmdAsArray, cmdAsArray);
      _exit(127);
   }
   else  //Parent process
   {
      bglist[index].pid = pid;  //Assign child pid to bglist
      strcpy(bglist[index].command, *cmdAsArray);  //Copy command to bglist
   }
}
/***bglistFunc***/
//Lists all background processes contained in bglist
void bglistFunc(bgjob_t bglist[]){
   clearList(bglist);   //Clear completed processes  
   int totalJobs;
   for(int index=0;index<MAX_BG;index++)
   {
      if(bglist[index].pid != INVALID_PID)  //Display active processes
      {
         int current = index + 1;
         printf("%d: %s\n",current,bglist[index].command);
         totalJobs += 1;
      }
      else  //Otherwise show as [empty]
      {
         int current = index + 1;
         printf("%d: [empty]\n",current);
      }
   }
   printf("Total Background Processes: %d\n",totalJobs);
}
/***bgkill***/
//Terminates a process with SIGKILL upon user specification
//of the process to be killed (stored in bglist)
void bgkillFunc(bgjob_t bglist[], char **cmdAsArray){
   clearList(bglist);   //Clear completed processes
   int index = atoi(cmdAsArray[0]);
   if(bglist[index-1].pid == INVALID_PID)  //If nothing in that slot, exit
   {
      printf("There are no processes to terminate at slot %d.\n",index);
   }
   else  //Otherwise, terminate process
   {
      kill(bglist[index-1].pid,SIGTERM);
      printf("Terminated process %d at slot %d.",bglist[index-1].pid,index-1);
   }
}
/***clearList***/
//Clears processes out of bglist if they've completed
void clearList(bgjob_t bglist[]){
   int status;
   for(int i=0;i < MAX_BG;i++)
   {
      if (bglist[i].pid != INVALID_PID)
      {
         if(waitpid(bglist[i].pid, &status, WNOHANG) != 0)
         {
            bglist[i].pid = INVALID_PID;   //Clear the process
         }
      }
   }
}
/***forkExecution***/
//Executes a fork operation that will handle the
//acceptance and execution of UNIX commands
void forkExecution(char**cmdAsArray) {
   int pid, status;
   pid = fork();
   if(pid < 0){
      //Fork operation did not succeed
      printf("The fork failed\n");
      exit(1);
   }
   if (pid == 0) {  //Child Process
      //Make UNIX commands available to the user
      if(execvp(*cmdAsArray,cmdAsArray) < 0){
         printf("Command Execution Failure\n");
         exit(0);
      }
   }
   else {  //Parent Process
      while(wait(&status) != pid);  //Waiting for child to complete
   }
}
