# coauthornetwork

## Introduction
This project consists of two parts: data gathering and data analysis. In data gathering process, which is highly time-consuming due to budget limit, I use both APIs and web-scraping to obtain author's data from Google Scholar, which include list of their co-authros, their h-index and number of citations they accumulate over years. There will be two sections in data analysis, one focuses on how likely authors with the similar h-index co-author with each other, the other address the questions whether authors with high citations and high h-index play more important role in the network. 

## Data Gathering
There are two separate sets of data needed for this project: network of co-author data and information about the authors data, including h-index and citations. The plan is to obtain network of co-author data, which would later be used to obtain information data. 

### Network of co-author data
There are quite a few resources that provide API for academic data, regarding publications, journals, authors, among others. Here are a few outlets that I tried along with their pros and cons, and the reasons why I ended up (not) using them:
* Open Alex API: Open Alex is an open source, with no API key requirement. Open Alex offers decent . However, I could not find 
* Scopus API: Scopus offers with generous quota (20000 calls per account). However, when I used Scopus API to obtain author data, the supposed-to-be-json file is not json file and can't be read by response.json command. I've tried different command like urllib.request but it did not work either. Furthermore, in ordert to get author result, I had to input both author name and their affiliation
* Scholarly, got error. Even the sample code did not work when running on my laptop
* SerpAPI is a third party that provides API for Google Search, including Google Scholar. However, SerpAPI limits 20 calls/hour and 100 calls/month. I circumvent around the system by creating multiple accounts using temporary emails. 

I started with one author: David Autor since I have reading and he is one of the key author of my field of interest: automation and its implication on employment. Furthermore, he is also a prominent scholar who co-authors with various stars in the field. David Autor's network should include key scholars in the field. 
After getting data of Daivd Autor's network, which is around 30, I scrape data of their co-author network, with the average of around per 15 each author. In the end I get around 3434 nodes, equivalent to 3434 unique authors. 

<p>

```python
#Function to get the scholar's co-authors
def cox(a, apikey):
    name = a.replace(" ","+")
    url = 'https://serpapi.com/search.json?engine=google_scholar_profiles&mauthors='+name+'&hl=en&api_key='+apikey
    response = requests.get(url)
    result = response.json()
    author_id = result['profiles'][0]['author_id']

    url ='https://serpapi.com/search.json?engine=google_scholar_author&author_id='+author_id+'&view_op=list_colleagues&api_key='+apikey
    response = requests.get(url)
    result = response.json()
    ls = []
    try:
        for i in range(len(result['co_authors'])):
            ls.append(result['co_authors'][i]['name'])
    except KeyError:
        pass
    return ls
GG = nx.add_nodes_from(all_co)
li = list(GG.nodes())
```
    
</p>

### Relevant information about the authors

Since SerpAPI does not provide with scholars' cummulative citations and h-index, I decided to scrape Google webpage for each author for such information. Google author ID is in google URL. I can access each author's google scholar webpage after obtaining their ID with the API call. 

<p>

```python
def info(a, apikey):
    name = a.replace(" ","+")
    url = 'https://serpapi.com/search.json?engine=google_scholar_profiles&mauthors='+name+'&hl=en&api_key='+apikey
    response = requests.get(url)
    result = response.json()
    author_id = result['profiles'][0]['author_id']

    url = 'https://scholar.google.com/citations?user='+author_id+'&hl=en'
    req = urllib.request.Request(url, headers = {'User-agent':'Modzilla/5.0'})
    webpage = urllib.request.urlopen(req)
    soup = BeautifulSoup(webpage, "lxml")
    h = []
    for a in soup.find_all("td"):
        h.append(a.string)
    citation = int(h[1])
    hindex = int(h[4])
    return author_id, citation, hindex
```
</p>


## Network Analysis
There are two approaches to the network analysis based on two hypotheses. First, I hypothesize that authors with high h-index or high citations would be more popular and important in the network, based on criteria like page-rank, centrality and betweeness. The first analysis would be finding the correlation-coeffecient between h-index/citations with other network indicies like page-rank or betweeness. Second, I hypothesize that authors with the same h-index or citations would tend to co-write a paper together. To test that hypothesis, there is a built-in function of networkx called ``` nx.numeric_assortativity_coefficient ```. To determine whether that is statistically significant, I would bootstrap the network with ``` nx.random_reference ```. However, since there are more than 3000 nodes in the network, each bootstrap takes 5 minutes. 100 boostraps would take about 8 hours. I would write the codes for the boostrap of 10 times, but might not be able to go through the process 1000 times. 

<p>

```python
el = []
for node,neighbors in all_co.items():
    for neighbor in neighbors:
        el.append((node,neighbor))
GG = nx.Graph()
GG.add_edges_from(el)

```
</p>

<p>

```python
#constructing dataframe
df_in = pd.DataFrame.from_dict(all, orient='index')
df_in.columns = ['ID','citations','h-index']

#constructing h-index and citations attributesof researchers of the network
nx.set_node_attributes(GG, values= df_in['citations'], name='citations')
nx.set_node_attributes(GG, values= df_in['h-index'], name = 'h-index')

```
</p>

<p>

```python
#Plotting the network
#Plotting labels (name of authors) for nodes with highest centrality
pos = nx.spring_layout(GG)

plt.figure(figsize=(100,80))

sizes = [ 20*GG.degree[node]+5 for node in GG.nodes]
dolphin_pr = nx.pagerank(GG)
colors = [ dolphin_pr[node] for node in GG.nodes]

pos = nx.spring_layout(GG)
s = nx.draw_networkx_nodes(GG, pos, node_size=sizes, node_color= colors,cmap='inferno')
cbar = plt.colorbar(s)
nx.draw_networkx_edges(GG, pos, edge_color='#4682B4')

#label top ten of centrality
centrality = dict(GG.degree())
degtop = sorted(centrality.keys(), key= lambda x:-centrality[x])[:50]
new_labels = {node:node for node in degtop}
nx.draw_networkx_labels(GG, pos, labels=new_labels, font_size=50)


plt.title('Mapping coauthorship', size = 50);
plt.axis('off');


```
</p>



### First analysis
Correlation between h-index, citations with pagerank, centrality and betweeness
<p>

```python
#Node centrality 
central = pd.DataFrame.from_dict(centrality,orient='index')
central.columns = ['centrality']
print(central['centrality'].nlargest(n=10))
print('correlation coefficient between h-index and betweenness is',df_in['h-index'].corr(central['centrality']))
print('correlation coefficient between h-index and betweenness is',df_in['citations'].corr(central['centrality']))
```
</p>

<p>

```python
#create Dataframe for pagerank
pagerank = pd.DataFrame.from_dict(dolphin_pr, orient='index')
pagerank.columns = ['pagerank']
print(pagerank['pagerank'].nlargest(n=10))
print('correlation coefficient between h-index and page rank is',df_in['h-index'].corr(pagerank['pagerank']))
print('correlation coefficient between h-index and page rank is',df_in['citations'].corr(pagerank['pagerank']))
```
</p>

<p>

```python
#Betweenness
between = nx.betweenness_centrality(GG)
betweenness = pd.DataFrame.from_dict(between, orient='index')
betweenness.columns = ['betweenness']
print(betweenness['betweenness'].nlargest(n=10))
print('correlation coefficient between h-index and betweenness is',df_in['h-index'].corr(betweenness['betweenness']))
print('correlation coefficient between h-index and betweenness is',df_in['citations'].corr(betweenness['betweenness']))

```
</p>
### Second Analysis
For the second analysis, I will calculate the assortativity coeffecient, which means that how likely two nodes with similar attribute connect with each other. Afterwards, I will boostrap the network to see if the correlation coefficient is statistically significant.
<p>

```python
c = nx.numeric_assortativity_coefficient(GG, attribute='h-index')
#boostrap the process
values = []
for i in range(10):
    R = nx.random_reference(GG)
    co = nx.numeric_assortativity_coefficient(R, attribute='h-index')
    values.append(co)
```
</p>

Plotting 
<p>

```python
v = pd.Series(values)
fig, ax = plt.subplots()
v.plot(kind = 'hist',  alpha = 0.75)

ax.tick_params(left = False, bottom = False)
for ax, spine in ax.spines.items():
    spine.set_visible(False)

quant_95 =  v.quantile(0.95)

#plotting the 95 percentile line
plt.axvline(quant_95, color = 'coral')

#plotting the result
plt.axvline(c, linestyle = ":")

plt.show();
```
</p>
