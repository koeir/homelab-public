# iwctl SSID Trouble
The wifi SSID I tried to connect to had special characters (Ñ and ×) that I couldn't type into the terminal

I tried entering Unicode with `Ctrl + Shift U`, but it wouldn't work
I tried entering using `Alt + 165`, but it wouldn't work
I assume the ISO I downloaded had a different layout or something.

I eventually tried redirecting the output of the network scan to a file, and then editing the file so that only the SSID remained.
Then, instead of typing the SSID, just cat the file for its SSID.

```
root@archiso ~ # iwctl station wlan0 get-networks > networks.txt
root@archiso ~ # vim networks.txt
root@archiso ~ # iwctl station wlan0 connect "$(cat networks.txt)"
```

And that seemed to work.
