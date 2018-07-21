# discord-webhook
Gachi is not gay gachi is 
https://discordapp.com/api/webhooks/459609169184555010/Js3YF1ZLxknoWJfUYPrTZHgHURGGzNS0DMIIK1alkUq9wN0c1v-MbU4lwLJ8nat2_hUh 
import sys
import logging
import os
import re
import time

import requests
import json
import filelock
from filelock import Timeout
import argparse

dir_path = os.path.dirname(os.path.realpath(__file__))
logger = logging.getLogger('debug')
logger.setLevel(logging.DEBUG)
handler = logging.FileHandler(filename=os.path.join(dir_path, 'debug.log'), encoding='utf-8', mode='a')
handler.setFormatter(logging.Formatter('%(asctime)s:%(levelname)s:%(name)s: %(message)s'))
logger.addHandler(handler)
lock = os.path.join(dir_path, '.lock')

try:
    if os.path.exists(lock):
        os.remove(lock)
except OSError:
    logger.info('Instance of the program is already running. Aborting')
    exit()

with open(lock, 'w'):
    pass

filelock = filelock.FileLock(lock)
try:
    filelock.acquire(timeout=1)
except Timeout:
    logger.info('Could not acquire lock file. Aborting')
    exit()


api = 'https://www.googleapis.com/youtube/v3/'
playlistItems = 'playlistItems'
everyone = re.compile('.*?@everyone.*?', re.DOTALL)
here = re.compile('.*?@here.*?', re.DOTALL)
default_message = 'https://www.youtube.com/watch?v={id}'
config = {}
parser = argparse.ArgumentParser()
parser.add_argument('--reload-on-empty-pages', action='store_true')
parser.add_argument('--reload-all', action='store_true')
parser.add_argument('--reload-specific', nargs='*', default=[])
args = parser.parse_known_args(sys.argv)[0]
reload_empty_pages = args.reload_on_empty_pages
reload_all = args.reload_all
reload_specific = args.reload_specific


# reload_empty_pages = True


def load_config():
    global config
    try:
        with open(os.path.join(dir_path, 'config.json'), 'r', encoding='utf-8') as f:
            config = json.load(f)
    except:
        logger.exception('Failed to load config')
        print('Failed to load config. Make sure the root folder has a file called config.json')
        exit(0)

    if 'api_key' not in config:
        print('api_key must be set to your youtube api key in config.json')
        exit(0)

    return config


def get(endpoint, params=None):
    url = api + endpoint + '?key=%s' % config.get('api_key')
    r = requests.get(url, params=params)
    return r


def get_all_pl_items(id, *part, page_token=False):
    page_token = page_token
    if reload_all or id in reload_specific:
        page_token = False
    recursion_stop = reload_empty_pages and page_token is False
    current_page_token = None
    params = {'part': ', '.join(part), 'playlistId': id, 'maxResults': 50}
    all_items = []
    while page_token is not None:
        if page_token:
            params['pageToken'] = page_token
        else:
            params.pop('pageToken', None)

        r = get(playlistItems, params=params)
        js = r.json()
        current_page_token = page_token
        page_token = js.get('nextPageToken')
        all_items.extend(js.get('items', []))

    if not all_items and reload_empty_pages and not recursion_stop:
        all_items, current_page_token = get_all_pl_items(id, *part)
        logger.info('Reloading playlist {} files'.format(id))
    return all_items, current_page_token


def get_new_vids(old, new):
    old_set = set(old.keys())
    new_set = set(new.keys())
    new_vids = new_set - old_set
    all_vids = {**old, **new}

    return {k: new[k] for k in new_vids}, all_vids


def post_webhook(message, webhook_):
    requests.post(webhook_,
                   json={'content': message},
                   headers={'Content-type': 'application/json'})


def format_vid_dict(items):
    _items = {}
    for i in items:
        id = i['snippet']['resourceId']['videoId']
        title = i['snippet']['title']
        _items[id] = title

    return _items


def format_messages(items, message_format=default_message):
    def format(msg):
        id, title = msg
        title = everyone.sub('@\u200Beveryone', here.sub('@\u200Bhere', title))
        return message_format.format(title=title, id=id)
    return list(map(format, items.items()))


def send_messages(messages_, webhook_):
    if messages_:
        for m in messages_:
            logger.info('Sending message %s' % m)
            time.sleep(4)
            try:
                post_webhook(m, webhook_)
            except:
                logger.exception('Error while posting webhook')
                time.sleep(10)
                try:
                    post_webhook(m, webhook_)
                except:
                    logger.exception('Could not post webhook message')


load_config()
if os.path.exists(os.path.join(dir_path, 'playlists.json')):
    with open(os.path.join(dir_path, 'playlists.json'), 'r', encoding='utf-8') as f:
        old = json.load(f)

else:
    # old = {'current_page': False, 'items': []}
    old = {}

logger.debug(f'Loaded {len(old.items())} old items')
new_vids = {}
new = old.copy()

for webhook in config.get('webhooks', {}).keys():
    webhook_cfg = config['webhooks'][webhook]
    playlist_ids = webhook_cfg.get('playlist_ids', [])
    part = ('id', 'snippet')
    message = webhook_cfg.get('message', default_message)
    all_old_items = {}
    all_new_items = {}
    for playlist_id in playlist_ids:
        if playlist_id in new_vids:
            all_new_items.update(new_vids[playlist_id])
            continue

        cached = old.get(playlist_id, {})
        last_page = cached.get('last_page', False)

        items, last_page_token = get_all_pl_items(playlist_id, *part, page_token=last_page)
        items = format_vid_dict(items)
        old_items = cached.get('items', {})
        new_items, all_list_items = get_new_vids(old_items, items)
        all_new_items.update(new_items)
        all_old_items.update(old_items)

        new[playlist_id] = {'last_page': last_page_token, 'items': all_list_items}
        new_vids[playlist_id] = new_items

    with open(os.path.join(dir_path, 'playlists.json'), 'w', encoding='utf-8') as f:
        json.dump(new, f, indent=4, ensure_ascii=False)

    new_items, all_items = get_new_vids(all_old_items, all_new_items)
    messages = format_messages(new_items, message)
    webhooks = webhook.split('\n')
    for wh in webhooks:
        if len(wh) < 10:
            continue

        send_messages(messages, wh.strip())
        time.sleep(3)

filelock.release()
if os.path.exists(lock):
    try:
        os.remove(lock)
    except OSError:
        logger.exception('Failed to clean up lock file')
