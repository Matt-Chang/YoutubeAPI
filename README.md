# YoutubeAPI

I used pandas to pull all the data of my favorite YouTuber, Ken Jee, from YoutubeAPI. And store the data in the AWS  RDS pgadmin database.



I first import the libraries. 

        import requests
        import pandas as pd 
        import time

I spend some time reading YouTube API documentation. I then found my API key and the channel ID of my favorite data scientist, Ken Jee. 

According to the YouTube API documentation, I use the URL” https://www.googleapis.com/youtube/v3/search? “combined with API_KEY, channel_id and other details you want to get. Details I want to get from Ken Jee’s video are video_id, video_title, video_data, and upload_date. I used for loop to make sure I include every information. And save them as the function get_video.
def get_video(df):

    pageToken = ''
    url ='https://www.googleapis.com/youtube/v3/search?key='+API_KEY+'&channelId='+channel_id+'&part=snippet,id&order=date&maxResults=10000'+pageToken
    response = requests.get(url).json()


    for video in response['items']:
      if video['id']['kind']=='youtube#video':
          video_id = video['id']['videoId']
          video_title = video['snippet']['title']
          upload_date = video['snippet']['publishedAt']
          upload_date = str(upload_date).split('T')[0]

Then the same approach to the video details, view_count, like_count favorite_count, and comment_count. And save them as the function get_video_details
 	def get_video_details(video_id):


    url_video_stats ='https://www.googleapis.com/youtube/v3/videos?id='+video_id+'&part=statistics&key='+API_KEY
    
    response_url_video_stats = requests.get(url_video_stats).json()

    view_Count = response_url_video_stats['items'][0]['statistics']['viewCount']
    like_Count = response_url_video_stats['items'][0]['statistics']['likeCount']
    favorite_Count = response_url_video_stats['items'][0]['statistics']['favoriteCount']
    comment_Count = response_url_video_stats['items'][0]['statistics']['commentCount']
        
    return view_Count,like_Count,favorite_Count,comment_Count

I write them into a panda dataframe using: 
view_Count,like_Count,favorite_Count,comment_Count = get_video_details(video_id)

        #save data in panda df
          df = df.append({'video_id':video_id,'video_title':video_title,'upload_date':upload_date,'view_Count':view_Count,'like_Count':like_Count,'favorite_Count':favorite_Count,'comment_Count':comment_Count},ignore_index=True)

Using this to build up the dataframe 


    df = pd.DataFrame(columns=['video_id','video_title','upload_date','view_Count','like_Count','favorite_Count','comment_Count'])
    df = get_video(df)

    Save them to csv for the next move: 

    df.to_csv('API.csv',encoding ='utf-8',index = False)
    df.to_excel('API.xls',encoding ='utf-8',index = False)

Import libraries: 

    !pip install psycopg2
    import psycopg2 as ps 

Get the host name, dbname, port, username, and password from AWS RDS I just created and connected them to the cloud. 

    host_name = ---------'
    dbname = ‘---------‘
    port = ‘-------‘
    username = '-----' 
    password = '-------'
    conn = None

    conn = connect_to_db(host_name, dbname, port, username, password)
    curr = conn.cursor()
    
I created a function connect_to_db  to see if my local device has been connected.

    def connect_to_db(host_name, dbname, port, username, password):
        try:
            conn = ps.connect(host=host_name, database=dbname, user=username, password=password, port=port)

        except ps.OperationalError as e:
            raise e
        else:
            print('Connected!')
            return conn

Table created and details inserted on the database with two functions: 

    def create_table(curr):
        create_table_command = ("""CREATE TABLE IF NOT EXISTS videos (
                        video_id varchar PRIMARY KEY,
                        video_title TEXT NOT NULL,
                        upload_date DATE NOT NULL DEFAULT CURRENT_DATE,
                        view_count varchar NOT NULL,
                        like_count varchar NOT NULL,
                        favorite_Count Integer NOT NULL,
                        comment_count varchar NOT NULL
                )""")
        curr.execute(create_table_command)

    def insert_into_table(curr, video_id, video_title, upload_date, view_count, like_count, favorite_Count, comment_count):
        insert_into_videos = ("""INSERT INTO videos (video_id, video_title, upload_date,
                            view_count, like_count, favorite_Count,comment_count)
        VALUES(%s,%s,%s,%s,%s,%s,%s);""")
        row_to_insert = (video_id, video_title, upload_date, view_count, like_count, favorite_Count, comment_count)
        curr.execute(insert_into_videos, row_to_insert)

Update new details to the database: 

    def update_row(curr, video_id, video_title, view_count, like_count, favorite_Count, comment_count):
        query = ("""UPDATE videos
                SET video_title = %s,
                    view_count = %s,
                    like_count = %s,
                    favorite_Count = %s,
                    comment_count = %s
                WHERE video_id = %s;""")
        vars_to_update = (video_title, view_count, like_count, favorite_Count, comment_count, video_id)
        curr.execute(query, vars_to_update)

    Check if the video exists and updates: 
    def check_if_video_exists(curr, video_id): 
        query = ("""SELECT video_id FROM VIDEOS WHERE video_id = %s""")

        curr.execute(query, (video_id,))
        return curr.fetchone() is not None
    def append_from_df_to_db(curr,df):
        for i, row in df.iterrows():
            insert_into_table(curr, row['video_id'], row['video_title'], row['upload_date'], row['view_count']
                            , row['like_count'], row['favorite_Count'], row['comment_count'])


    def update_db(curr,df):
        tmp_df = pd.DataFrame(columns=['video_id', 'video_title', 'upload_date', 'view_count',
                                    'like_count', 'favorite_Count', 'comment_count'])
        for i, row in df.iterrows():
            if check_if_video_exists(curr, row['video_id']): 
                update_row(curr,row['video_id'],row['video_title'],row['view_count'],row['like_count'],row['favorite_Count'],row['comment_count'])
            else: # The video doesn't exists so we will add it to a temp df and append it using append_from_df_to_db
                tmp_df = tmp_df.append(row)

        return tmp_df












