""" Returns the meeting room that matches the requirement """
import sys

def get_in_hour(meet_time, time_increment):
    """ Accepts start_time in HH:MM and chunk in mins. Returns added time in HH:MM format """
    meet_time_total = return_time_mins(meet_time) + time_increment

    hours = meet_time_total/60
    mins = meet_time_total%60
    return "{}:{}".format(int(hours), int(mins))

def return_time_mins(user_time):
    """ Accepts user_time in HH:MM format and returns equivalent mins from midnight """
    hours, mins = user_time.split(":")
    return int(hours)*60+int(mins)

def time_duration(end_time, start_time):
    """ Returns time difference in minutes between 2 given times """
    return return_time_mins(end_time)-return_time_mins(start_time)

def get_preferred_room(slot_window, all_matches):
    """ Accepts a Tuple of meeting rooms and list of all tuples, finds best meeting room """
    max_count = 0
    preferred = None
    for entry in slot_window:
        count = 0
        for slot in all_matches:
            if entry in slot:
                count += 1
        if count > max_count:
            max_count = count
            preferred = entry
    return preferred

def get_meeting_matching_slots(meeting_start_time, meeting_end_time, duration, matching):
    """ For a given timeslot, identifies and returning matching meeting room availability """

    all_matches = []
    for data_line in matching:
        data_elements = data_line.split(",")
        floor_room_number = data_elements[0]
        time_slots = data_elements[2:]

        while len(time_slots) > 0:
            start_time_slot = time_slots.pop(0)
            end_time_slot = time_slots.pop(0)
            slot_duration = time_duration(end_time_slot, start_time_slot)
            if slot_duration >= duration:
                if (return_time_mins(start_time_slot) <= return_time_mins(meeting_start_time) and
                        return_time_mins(end_time_slot) >= return_time_mins(meeting_end_time)):
                    all_matches.append(floor_room_number)
    return all_matches

if __name__ == '__main__':

    if len(sys.argv) != 2:
        print("ERR: Insufficient arguments.")
        print("Usage: ./nearest_meeting_room.py <team_size>,<team_floor>,<start_time>,<end_time>")
        sys.exit(1)

    TEAM_SIZE, TEAM_FLOOR_NUMBER, MEETING_START_TIME, MEETING_END_TIME = sys.argv[1].split(",")
    MEETING_DURATION = time_duration(MEETING_END_TIME, MEETING_START_TIME)

    MATCHES = []
    with open('./rooms.txt', 'r') as rooms_data:
        for line in rooms_data:
            MATCHES = list(filter(lambda x: x.split(",")[1] >= TEAM_SIZE, line.strip().split(" ")))

    h_possible_matches = get_meeting_matching_slots(MEETING_START_TIME, MEETING_END_TIME,
                                                    MEETING_DURATION, MATCHES)
    if len(h_possible_matches) > 0:
        h_nearest_floor = {}
        for possible_match in h_possible_matches:
            floor_number = possible_match.split('.')[0]
            floor_difference = abs(int(TEAM_FLOOR_NUMBER)-int(floor_number))
            h_nearest_floor[floor_difference] = possible_match
        NEAREST_FLOOR = (min(h_nearest_floor.keys()))
        print(h_nearest_floor[NEAREST_FLOOR])
    else:
        print("No single meeting room found for your selection")
        # CHUNK_DURATION = MEETING_DURATION*1/4
        # Hardsetting - Minimum chunk of 30minutes
        # TODO: Handle case where meeting room has more time available than chunk duration
        # e.g if meeting room has 45mins free then instead of taking for 30mins (chunk_duration), 
        # need to consume all 45mins and then look for other room from that point
        CHUNK_DURATION = float(30)
        print("Checking for meeting room in chunks of duration: ", CHUNK_DURATION)
        STARTING_SLOT = MEETING_START_TIME
        sub_slots = []
        while return_time_mins(STARTING_SLOT) < return_time_mins(MEETING_END_TIME):
            SUB_MEETING_END = get_in_hour(STARTING_SLOT, CHUNK_DURATION)
            sub_slots.append((STARTING_SLOT, SUB_MEETING_END))
            STARTING_SLOT = SUB_MEETING_END

        all_slot_matches = []
        for each_slot in sub_slots:
            h_possible_matches = get_meeting_matching_slots(each_slot[0], each_slot[1],
                                                            CHUNK_DURATION, MATCHES)
            if len(h_possible_matches) == 0:
                print("Sorry, cannot find sequence of meeting rooms for your selection")
                sys.exit(0)
            all_slot_matches.append(h_possible_matches)

        for index, each_slot in enumerate(all_slot_matches):
            preferred_room = get_preferred_room(each_slot, all_slot_matches)
            print(sub_slots[index], preferred_room)
