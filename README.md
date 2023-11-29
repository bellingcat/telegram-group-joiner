# Telegram Group Joiner
A web tool to automate joining groups, useful when you're entering a new research topic and have a start point of Telegram groups that are important to monitor. If you have private channel invites, those work too.


<h3 align="center">
  <a href="https://bellingcat.github.io/telegram-group-joiner/">bellingcat.github.io/telegram-group-joiner/</a>
  <br><br>
  example video <a href="https://github.com/bellingcat/telegram-group-joiner/blob/main/vid.mp4">here</a>
</h3>

<p align="center">
  <img src="https://github.com/bellingcat/telegram-group-joiner/assets/19508417/5e18ab3c-28a7-424f-a297-fb705b3358b2" width="500"/>

  <!-- <video src="https://github.com/bellingcat/telegram-group-joiner/raw/main/vid.mp4" width="500" height="400" controls> -->
</p>

You can preload a URL with links if you separate them by `;` in the URL using `links=` parameter:

```bash
# these 2 links
https://t.me/bellingcat
https://t.me/+privateInvite,123id
# should be put into the same line and separated by `;`
https://t.me/bellingcat;https://t.me/+privateInvite,123id
```
resulting in https://bellingcat.github.io/telegram-group-joiner/?links=https://t.me/bellingcat;https://t.me/+privateInvite,123id

### Development
The project uses tdlib for a full client-side telegram API experience, documentation of common methods is [here](https://core.telegram.org/tdlib/docs/classtd_1_1td__api_1_1_function.html).

```bash
# install dependencies
yarn

# run in development mode (hot reload)
yarn dev

# Compiles and minifies for production
yarn build
```
The project is auto-deployed to GitHub pages when changes to main are pushed.

(group ie channel ie chat)