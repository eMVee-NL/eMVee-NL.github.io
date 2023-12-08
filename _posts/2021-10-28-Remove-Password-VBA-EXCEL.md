---
title: How to remove a password in a MS Excel VBA project?
date: 2021-10-28 20:30:00 +0800
categories: [Tutorial, Password]
tags: [Excel, VBA, Password]
render_with_liquid: false
---

Years ago I automated some work in MS Excel with VBA. Sometimes I decided to protect the source code with a password. 
A password that later turns out not to provide sufficient protection for the file because the password can be removed. 
This was very nice for me, because I wanted to see the source code of a script I had made years ago. I did not remember the associated password.
The password on the MS Excel file to protect the source code was created by me so long ago that I could no longer remember it. 
And no, at the time I wasn't worried about ever wanting to look at the source code again. 
But through a training I was reminded that I had created something years ago and now I wanted to see the source code, but I couldn't access it because of the password.
_This post is purely for educational purposes and should not be misused._

## A password is not needed, we can remove it!
The password that protects a VBA source code in MS Excel can be removed in a number of steps. In the steps below I will share with you how you can remove the password from your password-protected file.
The steps below are tested in MS Excel 2019.

 1. If you don't have yet [7-Zip](https://www.7-zip.org/) and [Notepad++](https://notepad-plus-plus.org/downloads/) download them and install both applications. They are free to use.
 2. Open Explorer and navigate to the MS Excel file with the VBA code in it.
 3. Now we should press the right-click with the mouse on the xlsm file and choose `7-Zip` –> `Open archive`.
 4. We have to open the `xl` folder.
 5. Next we should drag the file `vbaProject.bin` from the 7-Zip environment (for example to your desktop). A copy is then made.
 6. Right-click with the mouse on the `vbaProject.bin` file and choose `Edit with Notepad++`.
 7. We should now search for `DPB` and replace it with `DPx`.
 8. Now you can save the file (this can be done with `CTRL + S`).
 9. Drag the `vbaProject.bin` file back into the 7-Zip folder and overwrite the original file.
 10. Open the Excel file. You will now receive a message that an invalid key has been found. Choose to continue loading.
 11. You will now receive an `unexpected error, message 40230`. Click `OK`.
 12. Save your Excel file again.
 13. Now the password for your VBA project is gone and you can view the project and the macro again.