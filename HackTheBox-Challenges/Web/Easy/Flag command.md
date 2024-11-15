# CHALLENGE DESCRIPTION
Embark on the "Dimensional Escape Quest" where you wake up in a mysterious forest maze that's not quite of this world. Navigate singing squirrels, mischievous nymphs, and grumpy wizards in a whimsical labyrinth that may lead to otherworldly surprises. Will you conquer the enchanted maze or find yourself lost in a different dimension of magical challenges? The journey unfolds in this mystical escape!

First, I entered the web using the provided IP and port. Upon entering, The web showed me this:

<div align="center">
  <img src=https://github.com/user-attachments/assets/8c3df340-c0ab-4d90-a4bd-9bbd8f492e51>
</div>

It looks like the web is some kind of game that we need to enter the given choices. Firstly, I tried to play the game and trial and error with choosing the choices. Until at the end, when it comes to the part where after choosing "SET UP CAMP", the next choices was all wrong.

Then, I tried to look for the web's source code and got 3 JavaScript file, **main.js**, **command.js**, and **game.js**. I looked into main.js and found this:

![image](https://github.com/user-attachments/assets/09b7c140-3d66-4b30-82d3-6a7d755f9198)

Apperently, the choices are set up only to SET UP CAMP. After that we can't do anything like I did before. But, it says that:

    
                if(data.message.includes('HTB{')) {
                    playerWon();
                    fetchingResponse = false;

                    return;
                }
So, if somehow I got to enter something that give a message that include `'HTB{`, I won. In the first if statement, which is this:

    if (availableOptions[currentStep].includes(currentCommand) || availableOptions['secret'].includes(currentCommand))

I found that this web will send a JSON data with `'secret'` in it. So, next thing I did was check for the request and response using developer tool. In the request for `file`, I found that it contain the choices and the secret. Which is:

![image](https://github.com/user-attachments/assets/a3d3db14-85fb-46e1-b4a9-379f6c581d33)

Next, I entered that secret and got the flag.

![image](https://github.com/user-attachments/assets/4c3f3400-c3e9-4d2a-a68c-3a6c4701fc8b)




