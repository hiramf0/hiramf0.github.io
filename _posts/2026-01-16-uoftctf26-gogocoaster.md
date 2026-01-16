## UofTCTF 2026 - Go Go Coaster! Writeup

"During an episode of Go Go Squid!, Han Shangyan was too scared to go on a roller coaster. What's the English name of this roller coaster? Also, what's its height in whole feet?

Flag format: uoftctf{Coaster_Name_HEIGHT}
Example: uoftctf{Yukon_Striker_999}

Author: levu12"

This is a writeup for this OSINT challenge for the UofTCTF 2026.

1. Let's look up on Google "Han Shangyan was too scared to go on a roller coaster."
- We are directed to <a href="https://www.cpophome.com/go-go-squid-yang-zi-li-xian/recap/12/">this site</a>, where we can see that we are referring to Episode 12.

2. Let's watch the episode itself now, here's a link to a YouTube video with the episode: <a href="https://www.youtube.com/watch?v=8N1k83-BzM4">Go Go Squid! EP12 | Yang Zi, Li Xian | CROTON MEDIA English Official</a>
- I skimmed through the video, hoping to see something like a roller coaster or an amusement park that the protagonist seem to be to scared to go into.

3. At 11:05, there seems to be an important landmark that could help me identify where the scene is taking place. I took a screenshot and used Google Images for more information.
- Success! It seems the park where the scene takes place is called "Happy Valley Shanghai".
- The roller coaster that we are looking for is at 11:25, it's a roller coaster with pink rails and blue steel beams.

4. Let's look up some attractions inside that park on YouTube: "happy valley shanghai attractions"
- This leads us to this video: <a href="https://www.youtube.com/watch?v=3ytWXZF9B0Q">Every Roller Coaster at Happy Valley Shanghai China!</a>

5. Minute 1:03 shows our attraction! "Diving Coaster"

6. Looking up "diving coaster happy valley shanghai" returns a Wikipedia article: <a href="https://en.wikipedia.org/wiki/Diving_Coaster">Diving Coaster</a>
- We can find the height in feet of the roller coaster here: 213ft

The flag is `uoftctf{Diving_Coaster_213}`.


