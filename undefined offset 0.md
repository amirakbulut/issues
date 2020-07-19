
I have a class that checks available timeslots. Which I use like:

    $timeslots = Calendar::create()
                ->withSlotDuration(30)
                ->withBusinessHours([
                    ["2020/07/17 09:00:00", "2020/07/17 13:00:00"],
                    ["2020/07/17 14:00:00", "2020/07/17 18:00:00"]
                ])
                ->withAppointments([
                    ["2020/07/17 15:00:00", "2020/07/17 15:30:00"],
                ])
                ->getSlots();

**Notice how the business hours include a 1 hour break**. This way the code works *correctly*.

However, when I use a day *without* work breaks, like:

    $timeslots = Calendar::create()
                ->withSlotDuration(30)
                ->withBusinessHours([
                    ["2020/07/17 09:00:00", "2020/07/17 18:00:00"],
                ])
                ->withAppointments([
                    ["2020/07/17 15:00:00", "2020/07/17 15:30:00"],
                ])
                ->getSlots();

It breaks with the error `Undefined offset: 0`  which refers to `$breaks[0][0]` in the `getAvailabilityArray()` function.

This is the related code:


    /**
     * Maps the end of each item to the start of the next one.
     *
     * @return array Returns an array which contains, in order, the new mapped array,
     */
    protected function mapArrayEndToStart($array): array
    {
        $result    = [];
        $itemCount = count($array);

        for ($i = 0; $i < $itemCount; ++$i) {
            $isLast = $i === $itemCount - 1;

            if ($isLast) {
                continue;
            }

            $result[] = [
                $array[$i][1],
                $array[$i + 1][0],
            ];
        }

        return $result;
    }

    /**
     * Gets an array containing the availability slots.
     */
    protected function getAvailabilityArray(): array
    {
        $breaks = [
            ...$this->mapArrayEndToStart($this->businessHours),
            ...$this->appointments,
        ];

        usort($breaks, fn ($a, $b) => $a[0] <=> $b[0]);


        $availability = [
            [
                $this->businessHours[0][0],
                $breaks[0][0],
            ],
            ...$this->mapArrayEndToStart($breaks),
            [
                $breaks[count($breaks) - 1][1],
                $this->businessHours[count($this->businessHours) - 1][1],
            ],
        ];

        return $availability;
    }

How to solve this? And why exactly is this happening?
