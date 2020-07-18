I have a PHP class, which generates available timeslots based on **business hours** and **existing appointments** you provide to it. You call it like this:

```
    $slots = Calendar::create()
        ->withSlotDuration(45)
        ->withBusinessHours([
            ['09:00:00', '13:00:00'],
            ['14:00:00', '18:00:00'],
        ])
        ->withAppointments([
            ['09:00:00', '09:10:00'],
            ['15:30:00', '16:15:00'],
        ])
        ->getSlots();

    return json_encode($slots);
```

This returns an array with all the available timeslots (each timeslot contains the format `[start, end]`), which is exactly what I want.

**However**, it only works with the format `h:m:s`, I need the script to work with `y-m-d h:m:s`, in order to specify on which day a specific appointment takes place.

This is the content of the class:

    <?php
    
    namespace App;
    
    use Carbon\Carbon;
    
    class Calendar
    {
        protected array $businessHours;
        protected array $appointments;
        protected array $availability;
        protected int $slotDuration;
    
        public function __construct()
        {
            $this->businessHours = [];
            $this->appointments  = [];
            $this->availability  = [];
            $this->slotDuration = 15;
        }
    
        /*
        |--------------------------------------------------------------------------
        | Fluent API
        |--------------------------------------------------------------------------
        */
    
        public static function create(): self
        {
            return new static();
        }
    
        public function withSlotDuration(int $slotDuration): self
        {
            $this->slotDuration = $slotDuration;
    
            return $this;
        }
    
        public function withBusinessHours(array $businessHours): self
        {
            $this->businessHours = $businessHours;
    
            return $this;
        }
    
        public function withAppointments(array $appointments): self
        {
            $this->appointments = $appointments;
    
            return $this;
        }
    
        /**
         * Gets the available slots.
         */
        public function getSlots(): array
        {
            $slots = [];
    
            foreach ($this->getAvailabilityArray() as [$start, $end]) {
                $slots = array_merge(
                    $slots,
                    $this->getSlotsFromPeriod(
                        Carbon::createFromTimeString($start),
                        Carbon::createFromTimeString($end)
                    )
                );
            }
    
            return $slots;
        }
    
        /*
        |--------------------------------------------------------------------------
        | Helpers
        |--------------------------------------------------------------------------
        */
    
        protected function formatDate(Carbon $date, string $format = 'H:i:s'): string
        {
            return $date->format($format);
        }
    
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
    
        /**
         * Gets an array of slots of the format [slotStartTime, slotEndTime] for the given time period.
         */
        protected function getSlotsFromPeriod(Carbon $start, Carbon $end): array
        {
            $count = (int) ($start->diffInMinutes($end) / $this->slotDuration);
            $slots = [];
    
            for ($i = 0; $i < $count; ++$i) {
                $slots[] = [
                    $this->formatDate($start),
                    $this->formatDate($start->addMinutes($this->slotDuration)),
                ];
            }
    
            return $slots;
        }
    }

What changes do I need to make, in order to work with `y-m-d h:m:s` format, instead of `h:m:s`?
