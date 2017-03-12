# search-engine
Web search engine implemented with Python

#STAGE 1
import urllib2
import random

def download_page(pagename):
    '''This downloads the pagename that was inserted in the crawl web function'''
    fullurl = 'http://cs.colgate.edu/cosc101/testweb/' + pagename
    fileobject = urllib2.urlopen(fullurl)
    return fileobject.read()

def extract_links(page_contents):
    '''this extracts the pagenames from the downloaded file object'''
    linklist = []
    index = 0
    while index >= 0:
        index = page_contents.find('<a href', index)
        if index >= 0:
            endquote = page_contents.find('"', index + 9)
            pagename = page_contents[(index + 9):endquote]
            linklist.append(pagename)
            index = endquote
    return linklist

def remove_tags(fulltext):
    '''This removes all the <> and everything in it (except it saves the pagenames
       within it'''
    
    no_tags = ''
    in_bracket = False

    for char in fulltext:
        if char == '<':
            in_bracket = True
            continue
        if char == '>':
            in_bracket = False
            continue
        if not in_bracket:
            no_tags += char
    return no_tags

def normalize_word(string):
    '''This converts all capital letters to lowercase and gets rid of spaces'''
    new_string = ''
    for char in string:
        if char.isalpha():
            new_string += char.lower()
    return new_string

def index_page(pagename, fulltext, myindex):
    '''Creates a dictionary that has words as keys and each page that they occur
       in as links'''
    for char in remove_tags(fulltext).split():
        key = normalize_word(char)
        if key not in myindex:
            myindex[key] = [pagename]
        else:
            if pagename not in myindex[key]:
                myindex[key].append(pagename)        

def crawl_web(page_name, reverse_index, webgraph):
    '''This starts from a.html and "crawls" through all of the links in the web'''
    page_content = download_page(page_name)
    links = extract_links(page_content)
    index_page(page_name, page_content, reverse_index)
    webgraph[page_name] = links
    for outlink in links:
        #print "visiting", outlink,"from", page_name
        if outlink not in webgraph:
            crawl_web(outlink, reverse_index, webgraph)

#------------------------------------------------------------------------
# STAGE 2
def random_surfer_simulation(web_graph,p,num_simulate):
    '''Creates dictionary that keeps track of each time a web page is hit and divides
       that amount by the value of num_simulate to get a float'''
    pages = web_graph.keys() #gets all the keys in the wer graph dictionary
    page_rank = {}
    for page in pages:
        if page not in page_rank:
            page_rank[page] = 0 #must be set to 0 so that the numbers that I
                                #accumulate later will be correct

    #This goes through the dictionary num_simulate times
    #and keeps track of each time it switches or stays with
    #the same key
    curr_page = random.choice(pages)
        
    for num in range(num_simulate):
        if random.random() > p:
            curr_page = random.choice(web_graph[curr_page])
            page_rank[curr_page] += 1
        else:
            curr_page = random.choice(pages)
            page_rank[curr_page] += 1

    #This divides the value obtained by the previous loop by the value that
    #num_simulate holds
    for values in pages:
        page_rank[values] = page_rank[values]/float(num_simulate)
    return page_rank
#------------------------------------------------------------------------
# STAGE 3
def list_union(list1, list2):
    '''This return a new list that contains all items found in either list, but
       it gets rid of duplicates'''
    new_list = []
    for items in list1+list2:
        if items not in new_list:
            new_list.append(items)
    return new_list

def list_intersection(list1, list2):
    '''Returns a new list that contains only those items that are found in both
       of the lists'''
    common = []
    for item in list2:
        if item in list1:
            common += [item]
    return common

def list_difference(list1, list2):
    '''This returns a new list that contains only those items that are found in
       the first but not in the second list'''
    new_list = []
    for item in list1:
        if item not in list2:
            new_list.append(item)
    return new_list

def get_query_hits(search, myindex):
    '''This returns a list of all the page names that match the search term'''
    if search not in myindex:
        return []
    else:
        return myindex[search]

def process_query(query, reverse_index):
    """
    Processes the type of query entered by the user based on the terms entered
    and returns a list of all the page names where that query is found.
    """

    if len(query) == 0:
        result = []
        return result
    
    query_word_list = query.split()
    inc = []
    exc = []
    intersect = False
    if query_word_list[0] == 'AND':
        intersect = True
        query_word_list = query_word_list[1:]
    for word in query_word_list:
        if word[0] == '-':
            target = exc
        else:
            target = inc
        word = normalize_word(word)
        if len(word) > 0:
            target.append(word)
    if not intersect:
        pages = []
        for word in inc:
            hits = get_query_hits(word, reverse_index)
            pages = list_union(pages, hits)
    else:
        pages = get_query_hits(inc[0], reverse_index)
        inc.pop(0)
        for word in inc:
            hits = get_query_hits(word, reverse_index)
            pages = list_intersection(pages, hits)
    for word in exc:
        hits = get_query_hits(word, reverse_index)
        pages = list_difference(pages, hits)
    return pages

def print_ranked_results(results, page_rank):
    '''This prints out the query results in rank order, from highest to lowest'''
    ranks = []
    for page in results:
        rank = page_rank[page]
        ranks.append((rank,page))
    ranks.sort()
    
    ranks.reverse()
    
    count = 1
    for j in ranks:
        print str(count) + ":",j[1],"(rank:",str(j[0]) + ')'
        count += 1

def search_engine(revindex, page_rank):
    '''This asks the user for a search term, processes it, and prints out the
       ranked results'''
    search = raw_input("Search terms? (enter 'DONE' to quit) ")
    while search != 'DONE':
        results = process_query(search, revindex)
        if results == []:
            print "No matches for search terms."
        print_ranked_results(results, page_rank)
        search = raw_input("Search terms? (enter 'DONE' to quit) ")
        
def main():
    '''This brings everything together and crawl_web must be called first, then
       random surfer, then the search engine'''
    print 'Welcome to my mini search engine!'
    p = 0.15
    num_simulate = 100000
    revindex = {}
    webgraph = {}
    crawl_web('a.html', revindex, webgraph)
    page_rank = random_surfer_simulation(webgraph, p, num_simulate)
    search_engine(revindex, page_rank)
main()
