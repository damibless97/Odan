Files
-----

The default data directory on Windows has been moved from `C:\Users\Username\AppData\Roaming\Odan`
to `C:\Users\Username\AppData\Local\Odan`. Odan Core will check the existence
of the old directory first and continue to use that directory for backwards
compatibility if it is present.