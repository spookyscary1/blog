---  
author:  
 name: "spook"  
date: 2022-06-22  
linktitle:  "Tool touch up : web content scanner 1"
type:  
- post  
- posts  
title:  "Tool touch up : web content scanner 1"
weight: 10  
series:  
- tool touch up 
---

I noticed the web content scanner I wrote in go for [this blog post](https://spook.pw/posts/2022/05/dawn-writeup-but-with-slightly-more-programming/) does not work on websites using HTTPS when the TLS certificate cannot be validated. I decide to make some improvement to the code and generally refactor it. I initially wanted to add a flag to skip validating SSL/ TLS certificates. After mulling that over for a bit, I decided that adding a flag was not needed. Skipping the validating of certs can be the default behavior. Similar tools seem to do so also. 

The first order of business is to set up a web server using HTTPS with self-signed certs. I used [updog](https://github.com/sc0tfree/updog0) to set up an appropriate webserver. 
```bash 
>updog --ssl
```
I can attempt to run the tool to verify the problem exists. 
```bash 
> go run directory_scanner.go -h  https://192.168.1.130:9090/ -w list  
2022/06/14 12:50:45 Get "https://192.168.1.130:9090/dkjaskfsjd": x509: cannot validate certificate for 192.168.1.130 because it doesn't contain any IP SANs  
exit status 1

```

Documentation on the [http package](https://pkg.go.dev/net/http) proved to be useful. Making a transport allows for control over compression, TLS configuration, proxies, and keep alives. Since clients are safe for reuse, I moved that section of code out of my for loop. I do not need to repeatedly create clients. The code now looks like so:

```go
...
// creating a client that skips verifying tls certs

tr := &http.Transport{

TLSClientConfig: &tls.Config{InsecureSkipVerify: true},

}

client := &http.Client{Transport: tr}

  

for scanner.Scan(){

  

resp, err := client.Get(host +scanner.Text())
...
```
I run the program to verify the fix works. 

```bash 
> go run directory_scanner.go -h  https://192.168.1.130:9090/ -w list  
https://192.168.1.130:9090/dkjaskfsjd  
https://192.168.1.130:9090/adskljfakldf  
https://192.168.1.130:9090/asdklfjadf  
https://192.168.1.130:9090/daklfjda  
https://192.168.1.130:9090/dsfasdfj  
https://192.168.1.130:9090/adsffa
```
It does. Updog returns a 302 HTTP response status code when a file does not exist. I now realize it would be very nice to display which HTTP response code was returned by the webserver. Making that change was simple.
```go
fmt.Print(host +scanner.Text()+" ")

fmt.Println(resp.StatusCode)
```
I then decided to make not following redirects the default behavior. I had to change changes to the client. The checkredirect function is overridden with a function that returns the request instead of following the redirect. 
```go 
client := &http.Client{

Transport: tr,

CheckRedirect: func(req *http.Request, via []*http.Request) error { return http.ErrUseLastResponse },

}

```
Running to program verifies that the redirects are not followed. 
```bash 
> go run directory_scanner.go -h  https://192.168.1.130:9090/ -w list  
https://192.168.1.130:9090/dkjaskfsjd 302  
https://192.168.1.130:9090/adskljfakldf 302  
https://192.168.1.130:9090/asdklfjadf 302  
https://192.168.1.130:9090/daklfjda 302  
https://192.168.1.130:9090/dsfasdfj 302  
https://192.168.1.130:9090/adsffa  302
```
I now think the program could be improved by allowing a user to specify which status code they care about. I figured the user could input a comma seperated list of HTTP status codes to be displayed. Alternatively, the user could enter codes they wish to ignore. I started by writing the function to parse the comma seperated list into a useful data type to be used internally by the tool. I stored the list of status codes inside a map because the internet told me they behaved like dictionaries, and my main use of the list will be checking if items exist in the list. I believe this should be faster with an dictionary than an array. I ended up with code that looks like so:
```go
func parseCodeList (list string) (map[int64]bool, error) {
array := strings.Split(list,",")
fmt.Println(array)
var codes= make(map[int64]bool)
for i := 0; i < len(array); i++ {
x, err := strconv.ParseInt(array[i], 10, 64)
if err != nil {
return codes, errors.New("invalid comma seperated status code list")
}
codes[x]=true
}
fmt.Println(codes)
return codes, nil

```
The function turned out simple enough. I dived back into researching the flag library to settle on how I wanted the flags to work. I decided there would be a flag to pass in a list of status codes and an optional flag that decides if the program should use that only allow status codes in that list or only ignore status codes on that list. I only had to add an if statement to ensure our flag was the valid value. 
```go 
// sanity check code behavior varaiable

if !(strings.ToLower(codeBehavior)=="allow" || strings.ToLower(codeBehavior)=="deny" ){

fmt.Println("Error: b must be set to allow or deny")

os.Exit(4)

}

```
It just converts the user input to lowercase so that the comparison is case insensitive and then ensures the input is either allow or deny. I can now begin working on actually implementing the use of the optional status code list. I have two cases where I want to print an URL to the screen. I wound up with the code below. 
```go 
_, exists := mapCodeList[resp.StatusCode]

  

if codeBehavior=="allow" && exists== true{

fmt.Print(host +scanner.Text()+" ")

fmt.Println(resp.StatusCode)

} else if (codeBehavior=="deny" && exists==false) {

fmt.Print(host +scanner.Text()+" ")

fmt.Println(resp.StatusCode)

}
```

That finishes up all the new features I was hoping to implement. I now can have fun cleaning up the code so it doesn't look like an unreadable mess filled with assorted debug print statements. I created a function that printed URL and response codes to the console for code duplication purposes. The finished code can be found [here](https://github.com/spookyscary1/directory-scanner)