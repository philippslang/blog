site/
hugo new posts/my-first-post.md
hugo serve -D
rm -rf public
hugo -D

./
firebase deploy --only hosting

TODO
----
check into auto deploy option in firebase
consistent file-directory naming in posts-static/img