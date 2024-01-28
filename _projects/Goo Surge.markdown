---

layout: project
title: Goo Surge (Pirate Software Game Jam 2024)
thumbnail: \assets\gooSurge.png
videolink: youtube.com
shortdescription: 2 week game jam project in Unity, made in a team with my coursemates, utilising Compute Shaders.
permalink: /Goo-Surge
priority: 2
---

<h1>Play the Game</h1>
<iframe frameborder="0" src="https://itch.io/embed/2492552?link_color=d55bfa" width="552" height="167"><a href="https://nijelous.itch.io/goo-surge">Goo Surge by Nijelous, JunkyX1122, HeavyMetalGameDev, benblack90</a></iframe>
<h1>Summary</h1>
Goo Surge was a game made in two weeks by me and 3 of my coursemates on the Newcastle University Games Engineering Course. We had a large gap between Christmas and returning to university, so we decided to make productive use of our time and do a game jam together. We decided upon the Pirate Software Game Jam as the dates lined up well, and doing a 2 week game jam gave us much more time to get used to working with eachother and setting up the project.

When starting the jam, we agreed that we wanted to focus on making something interesting from a technical standpoint, and to challenge ourselves to do something we hadn't programmed before.

<h1>Gameplay</h1>
The player controls a wizard who has been tasked to break out the imprisoned goo, steal schematics and destroy as much of the facility as they can.
Our main gameplay mechanic was the goo. In our game, there is an ever expanding pool of goo, which spreads throughout the level, and the player can alter its properties by either heating it up (making it faster and more destructive) or cooling it down (making it slower and safer). When the goo is hot, it can destroy objects and walls in the level, giving the player points and opening up new areas of the level, but it can also hurt the player, so they need to be careful.

![image](\assets\gooCold.png)
<h6>Goo that has been cooled, making it expand slower</h6>

<h1>Drawing runes</h1>
The supporting mechanic of our game was rune drawing. In order to cast a spell, the player must open their canvas and draw the corresponding rune correctly. This adds an extra element of difficulty to the game and makes the game feel more frantic.

![image](\assets\spell.gif)
<h6>Casting different spells is done by drawing different symbols.</h6>
The rune drawing mechanic was not programmed by me, instead being made by <a href = "https://alexfalldev.wordpress.com/">Alex (click for link)</a>. Because of this, I do not know all the technical details on how it works.
From a gameplay standpoint, the mechanic was a good addition to the game, as it added some extra challenge and tension. However, it also added frustration for the player when they thought they drew a rune correctly, only for the game to not recognise it.


<h1>The Goo</h1>
<h2>Data</h2>
We decided early on that the main way the player would interact with the goo would be by heating it up. As such, we knew we needed a way to both store the temperature on a per pixel basis, and to run code on each tile of goo every time we wanted to update it. With a technical challenge such as this, we agreed that we had do these calculations on the GPU, as this was a very parallel problem. Therefore, we decided to attempt to do this using Compute Shaders, something none of us had never done before.
My and my teammate <a href = "https://benschwarz90.wixsite.com/portfolio">Ben (click for link)</a> were responsible for the goo mechanics in the game, whilst our other team members worked on the Rune Drawing and Tile Grid mechanics respectively.
To store the information for the goo, we decided to use a texture instead of a buffer. This was for a few reasons.

Firstly, textures are easy data containers to use with shaders, as there is very little manual setup: Unity's Compute Shaders by default use RenderTextures. Secondly, we knew that the number of types of data would match up with the number of texture channels: one channel for the type of goo, one channel for temperature of the goo, one channel to track how long until a goo tile spreads, and one channel for a target temperature for the goo to reach over time.
```cs
public enum GridChannel
{
    TYPE,TEMP,GOOAGE,TARGET_TEMP
}
```
<h6>Enum used when interacting with different channels of the goo texture.</h6>

This did mean however that we were wasting bandwidth slightly when sending a texture. For example, the "TYPE" channel should only ever have 1 of 4 values, and therefore storing it in a float (the type that our texture used for it's channels) is slightly wasteful. We decided this tradeoff was worth it for the ease of use.
```cs
public enum GridTileType
{
    BLANK,GOO_SPREADABLE,GOO_UNSPREADABLE,STATIC,MAX_TYPE
}
```
<h6>Enum used for the different types of pixels.</h6>

The final big advantage was that it would be very easy to render the goo to the screen. Since the goo data is stored in a texture, all we needed to to was to apply that texture to a quad, and create a custom fragment shader to convert from goo data into a visual representation. To do this, I made a new fragment shader that will interpolate between three colours: one hot, one normal, and one cold. The temperature value is squared such that changes in temperature are shown more drastically when the goo is hot. Additionally, any pixel that has the type "STATIC" or "BLANK" will have the alpha value set to 0, so when rendered, the pixel is not visible.
```hlsl
    if (data.y <= 0.5f)
    {
        colour = lerp(coldTemp, normalTemp, (data.y * 2) * (data.y * 2));
    }
    else
    {
        colour = lerp(normalTemp, hotTemp, ((data.y - 0.5f) * 2) * ((data.y - 0.5f) * 2));
    }
```
<h6>Snippet from the fragment shader used to convert from goo data to visuals. data.y is the temperature value of the pixel.</h6>

We could have instead used a Compute buffer (which we originally started using), but switched to a texture relatively quickly.

Instead of calling the shader every frame (which could very easily become inconsistent) we decided to utilise Unity's coroutine system, to have a coroutine that updates the goo every 0.05 seconds.
Unity's support for compute shaders was very intuitive, and this was our function to dispatch the compute shader:
```cs
    IEnumerator UpdateGoo()
    {
        WaitForSeconds wfs = new WaitForSeconds(0.05f);
        cs.SetInt("aspectX", xSize);
        cs.SetInt("aspectY", ySize);
       
        while (active)
        {
            cs.SetTexture(0, "Result", renderTexture);
            cs.Dispatch(0, renderTexture.width / 16, renderTexture.height / 16, 1);
            GetGooDataFromGPU();
            yield return wfs;
        }
    }
```
<h2>Spreading goo</h2>
To make the goo spread, we made some rules, in a similar way to Conway's Game Of Life:
- Goo can only spread to empty pixels.
- Goo can only spread when its "GOOAGE" is at maximum.
- Only pixels that are "GOO_SPREADABLE" can spread.
- Once a pixel has spread, it becomes "GOO_UNSPREADABLE".
- Hotter pixels spread faster than cold ones.
- When spreading, new goo pixels will have a slightly more normal temperature than the pixel it spread from.

All of this logic was handled in one compute shader, which applies these rules to every pixel each time the shader is ran.
```hlsl
void Spread(uint3 id)
{
    float4 values = Result[id.xy];
    values.x = 2.0f/255.0f; //Set type to GOO_UNSPREADABLE
    Result[id.xy] = values;
    
    values.x = 1.0f/255.0f;//Set type to GOO_SPREADABLE
    values.z = 0.0f;
   
    values.y -=  1.0f *values.x; //Reuse values.x and reduce temperature
    
    float2 temp = id.xy;

    //pixel to the right
    temp.x += 1; 
    if (temp.x < aspectX && Result[temp.xy].x == 0)
    {
        Result[temp.xy] = values;
    }

    //pixel to the left
    temp.x -= 2;
    if (temp.x >= 0 && Result[temp.xy].x == 0)
    {
        Result[temp.xy] = values;
    }
    
    //pixel above
    temp.x += 1;
    temp.y += 1;
    if (temp.y < aspectY && Result[temp.xy].x == 0)
    {
        Result[temp.xy] = values;
    }
    
    //pixel below
    temp.y -= 2;
    if (temp.y >= 0 && Result[temp.xy].x == 0)
    {
        Result[temp.xy] = values;
    }
}
```
<h6> Snippet of compute shader code to spread goo to 4 adjacent tiles, with slightly lower temperature.</h6>

The reason that we have the value `2.0f/255.0f` in here is due to conversions that occur between the CPU and GPU: on the CPU, our values range from 0-255 per channel, but the GPU deals with numbers normalised to be in the range `0.0f`-`1.0f`. Therefore, any calculations we do in the shader that we want to be the same on the CPU must be divided by `255.0f` to get them in the range `0.0f`-`1.0f`.
<h2>Balancing temperature</h2>
Spreading goo is an important mechanic, but with just this, the temperature of a goo pixel would never change over time. Furthermore, if the player heats up a pixel of goo, then goo tiles around it should also heat up. We therefore had a secondary EntropyShader.compute which handles distribution of heat and the cooling down of goo.

<h2>Checking for Goo</h2>

<h1>Performance limitations</h1>
Our main performance bottlenecks were not actually related to the speed of execution of the compute shaders: these had very little impact on the game. The main problem was sending data between the GPU and the CPU, as there needed to be consistency between the CPU side texture and the GPU side texture. The reason for this is that although all goo spreading and temperature logic was handled in the Compute Shader (GPU) any objects in the game such as walls, desks or tables needed to be able to check for hot goo, such that they could take damage.

Throughout the project, I used Unity's profiler regularly to find which functions and parts of our code were being particularly slow.
For example, I noticed that when first releasing the goo, the game would stutter quite badly. Upon looking at this in the profiler, the cause of the issue was actually the audio:



<h1>What went well</h1>
On a technical level, I am very happy with what we managed to achieve as a team.
<h1>What I would improve</h1>
Feedback from people who played the game seemed to be mixed, with most people commenting on the interesting mechanics and liking the overall style, but finding the game to be too confusing or the goals to be poorly communicated. I definitely agree that the aim of the game was not clear enough, and the game should definitely had a tutorial, or at least more instruction on what to do. If I could go back, I would add an arrow on the UI to point the player to where they need to go.

